# File Transfer

## Model and ordering

The sender starts with regular files selected on the local machine and opens them in a way that makes obvious changes easier to detect. The first `TransferRequest` is intentionally modest: safe display names, file count, and sizes. It does not include file contents or the sender's absolute paths.

After the pairing-authenticated session is confirmed, the sender provides one `FileMetadata` record per file, including the exact length and an optional precomputed SHA-256 digest. The receiver validates the whole manifest and prepares its destination before it sends `TransferReady`.

Version 1.0 transfers files one at a time in manifest order. That is less ambitious than multiplexing, but it keeps flow control, disk policy, progress, and failure handling understandable. I can add parallel files later if measurements justify the extra states.

## Metadata and names

Each file has a unique file ID, display name, exact unsigned 64-bit length, ordinal, and SHA-256 when available. Optional media type and modified time are informational and MUST NOT drive execution or integrity. Sender-local paths, owners, permissions, ACLs, extended attributes, and symlink targets are excluded initially.

The receiver:

1. Treats the name as an untrusted single-component suggestion.
2. Sanitizes/displays it without terminal controls.
3. Applies destination OS rules and local collision policy.
4. Maps the file ID to a locally chosen final name under a user-selected root.
5. Never returns the local destination path to the sender unless a future privacy-reviewed feature needs it.

Invalid UTF-8 names cannot be represented in the initial interoperable schema. The sender must generate a safe replacement display name and may indicate `name_was_lossy=true`; a future opaque-byte name capability must define OS mappings. Directories/deep paths are unsupported.

## Source consistency

Open and inspect each regular file before the request. Keep a stable handle where practical. Before and after streaming, compare file identity, type, size, and relevant modification metadata. If it changes, cancel that file/transfer with `FileUnavailable` rather than silently sending a mixed version. True snapshot semantics need filesystem-specific support and are future work.

Zero-byte files are valid: send metadata, no `FileChunk`, then `FileComplete`; receiver hashes the empty byte string and finalizes an empty file.

## Chunking and streaming

- Default negotiated payload: 256 KiB; legal range 64 KiB–1 MiB and never above data-frame maximum.
- Read one or a small bounded number of chunks ahead. Never load an entire file or manifest payload into memory.
- Each chunk identifies transfer ID, file ID, zero-based offset, data length, bytes, and optional end hint. The protected session sequence supplies replay/order protection.
- Initially require contiguous monotonically increasing offsets per file. Duplicate or gap is `InvalidMessage`/`UnexpectedMessage` and fails the transfer.
- Receiver validates `offset + length` with checked arithmetic and against declared file length before write.
- Hash exactly accepted bytes while streaming to the temporary file.

## Flow control, acknowledgement, and backpressure

| Strategy | Benefit | Cost |
| --- | --- | --- |
| Every chunk | Precise application progress/checkpoint | RTT chatter and poor throughput if stop-and-wait |
| Ranges | Handles out-of-order/parallel data | More complex state; unnecessary for sequential TCP 1.0 |
| Cumulative ACK | Compact highest contiguous offset | Cannot describe holes, which 1.0 forbids |
| Transport reliability only | Minimal protocol overhead | No application confirmation that disk/write/hash advanced |
| Checkpoint ACK | Bounded sender window and useful progress | Some duplicate state atop TCP |

My working choice is to let TCP/TLS handle retransmission while Lanweave reports application progress with cumulative `ChunkAcknowledgement` messages. The receiver sends one after 4 MiB of accepted data, after one second, or at the end of the file. An 8 MiB default receive window bounds how far ahead the sender may run, and ordinary socket backpressure may make that window smaller.

An acknowledgement means that Lanweave accepted the bytes, not necessarily that the operating system has made them durable. Version 1.0 should report `durable=false` until I have chosen and documented an fsync policy.

Cancellation/control frames receive priority over queued chunks. Receiver sends updated window or zero window if disk is temporarily blocked, subject to inactivity timeout.

## Hashing strategy

| Time | Advantage | Cost/limitation |
| --- | --- | --- |
| Before send | Receiver knows expected hash before bytes; catches source changes against later stream | Reads huge file twice; delays request and may become stale |
| During send | No extra sender read; hashes exact transmitted bytes | Expected hash known only at `FileComplete` |
| Receiver during write | No extra receiver read; hashes exact accepted stream | Depends on correct streaming implementation |
| Receiver after write | Independent second pass catches some I/O/logic issues | Doubles disk reads and delays completion |

For the first implementation, the sender hashes while it reads and puts the final digest in `FileComplete`. The receiver hashes while it writes and compares the result before it gives the file its final name. A small file, or a file with a trustworthy cached digest, may also include the hash in `FileMetadata`; if it does, `FileComplete` must repeat the same value.

I do not want to make every large transfer wait for a complete pre-read. A later high-assurance mode may read the temporary file again after flushing, but the UI should be honest about the extra I/O.

SHA-256 provides content integrity when delivered inside the authenticated session; by itself it does not authenticate the sender.

## Destination and finalization

- Preflight total declared bytes plus safety margin and local quota before `TransferReady`; recheck/enforce during writes.
- Create each temp file in the final destination directory with unpredictable local name, restrictive permissions, and create-new/no-follow behavior.
- Default collision policy is reject or locally rename (for example `name (1).ext`) after explicit UI display. Never silently overwrite.
- An approved overwrite writes a new temp file and atomically replaces only after verification; platform semantics and backup policy require testing.
- After length/hash success, flush as configured, close safely, atomically rename on the same filesystem, then mark file complete.
- A multi-file transfer is not globally atomic. If a later file fails, already finalized files remain and UI reports partial completion; initial policy may instead delay all renames, at greater disk/cleanup complexity. This is **Needs Research**.

## Failure and cleanup

On cancellation, network loss, timeout, write error, excessive bytes, or hash mismatch, stop accepting chunks, close handles, and delete partial temp files by default. A cleanup error is logged locally with a redacted path. Partials are not exposed under final names. Initial version has no resume; keeping partials provides little value and increases privacy/storage risk.

Insufficient storage before readiness yields `InsufficientStorage`; during transfer it aborts the entire transfer. Illegal names yield `InvalidFilename` before readiness. A missing/changed sender source yields `FileUnavailable`. Hash mismatch yields `HashMismatch` and no finalization.

## Progress

Per-file progress is accepted contiguous bytes / declared bytes; transfer byte progress is sum of accepted bytes / declared total. Zero-total transfers display file-count progress and complete normally. UI must label sender “sent/acknowledged” separately from receiver “verified/finalized.” Do not exceed 100% if malicious counts arrive; validation should have rejected them.

## Resume and parallelism

Resume, content-addressed chunks, sparse-file support, range acknowledgements, and parallel files are not part of version 1.0.

When I design resume, it will need authenticated durable checkpoints, a way to prove the source file is still the same, clear ownership of temporary files, chunk hashes, expiry, and fresh session keys and nonces. Reusing an old session ID is not a resume protocol.
