# File Transfer

This document gives normative file-handling detail for experimental protocol version 1. Message schemas and framing are defined in [Message Format](MESSAGE_FORMAT.md); connection sequencing and security requirements are defined in [Protocol](PROTOCOL.md).

## Pairing prerequisite

Pairing MUST complete before transfer disclosure or approval. After TLS and `hello`, the peers complete the fixed SPAKE2 profile, including mutual confirmation and binding to the live TLS exporter. The sender MUST NOT send `transfer_request`, a manifest, or any filename, size, or file count before pairing succeeds.

Pairing authenticates the live channel, not a transfer the receiver has not yet seen. After pairing, the sender sends exactly one `transfer_request`; the receiver then parses it within the fixed wire maxima, validates and displays the complete manifest, obtains separate explicit approval, and prepares the destination. Lower local file-count, total-size, storage, and resource policies are applied only during this post-pairing admission. Preparation or policy failure produces a rejecting `transfer_response`. Only after successful preparation does the receiver send an accepting `transfer_response`, immediately followed by `ready`; DATA remains forbidden until `ready`.

## Manifest

The sender MUST build the manifest locally from 1 through 1,024 user-selected regular files. These are fixed wire/parser bounds; a receiver applies any lower local policy only during post-pairing manifest admission. Local construction may occur before pairing, but no part of it may be transmitted before pairing. The sender MUST reject directories, symlinks, special files, and anything it cannot represent as a single filename component. Each immutable entry contains only:

- `name`: non-empty UTF-8, at most 255 encoded bytes, with no path separators or control characters;
- `size`: the exact byte length, from zero through `2^53-1`.

Array position is the file's zero-based index and transfer order. Count and total size are derived with checked arithmetic. There are no file IDs, explicit ordinals, paths, timestamps, permissions, media types, or pre-transfer hashes, and there is no second metadata phase.

The sender SHOULD open and inspect each source before requesting the transfer and keep a stable handle where practical. Before sending a file, it MUST confirm that the source is still a regular file with the declared size. If it cannot send the separately approved manifest unchanged, it MUST send terminal `cancel` rather than edit the manifest.

## Name admission

Names are untrusted filename components received only after authenticated pairing, not destination paths. Pairing does not make them safe and does not imply consent. Before sending an accepting `transfer_response`, the receiver MUST apply its actual destination-platform validity and equivalence rules, including normalization and case behavior where applicable. It MUST reject the entire manifest if:

- any name is invalid;
- two requested names are platform-equivalent;
- a requested name is platform-equivalent to an existing destination;
- it cannot safely plan every final name and required resource.

The receiver MUST NOT sanitize into a different name, auto-rename, or overwrite. User consent applies to the displayed complete manifest exactly as sent. Acceptance is all-or-nothing.

## Destination handling

After local approval but before sending an accepting `transfer_response`, the receiver MUST prepare the destination for the exact manifest. Preparation includes selecting and securely opening the destination root, rechecking name and collision rules, admitting storage and local resource policy, and creating the current temporary file under receiver control, preferably in the final destination directory, with an unpredictable name, create-new and no-follow behavior, and restrictive permissions. If any preparation step fails, the receiver sends a rejecting `transfer_response` and MUST NOT report acceptance.

After preparation succeeds, the receiver sends the accepting `transfer_response` followed by `ready`. The receiver MUST NOT wait for DATA or perform another fallible preparation phase between those messages. Temporary names and local destination paths are never sent to the peer.

The receiver MUST write only to the current temporary file. After successful length and hash verification, it MUST close or flush according to local durability policy and finalize with an atomic no-replace operation where the platform supports one. If safe no-replace finalization cannot be guaranteed, the file fails. Lanweave never overwrites an existing file in version 1.

Multi-file transfer is not globally atomic. Each verified file is final before the next starts. If a later file fails, the verified prefix remains.

## Sequential data

After `ready`, both peers set the current index to zero. The sender sends zero or more raw DATA frames for that file, with one or more required for a non-empty file. Each DATA frame contains 1 through 1 MiB bytes. DATA carries no index or offset; stream order and state identify its file and position.

The receiver MUST reject a DATA frame outside the receiving-file state or larger than the current file's remaining declared bytes. It increments the current byte count and SHA-256 state only for bytes successfully written to the current temporary file. Neither peer may advance to another index independently.

TCP/TLS supplies reliable ordering, retransmission, and backpressure. Implementations MUST bound socket, file-I/O, and outbound queues and SHOULD read only a small bounded amount ahead. Version 1 adds no chunk ACK or application flow window.

For a zero-byte file the sender sends no DATA. It immediately sends:

```json
{"type":"file_end","index":0,"sha256":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"}
```

## Verification

The sender computes SHA-256 over exactly the bytes read into DATA frames. After it has sent exactly the declared size, it sends `file_end` for the current index and digest. It MUST then wait without sending DATA for another file.

The receiver accepts `file_end` only for the current index. It compares the received byte count with the manifest size and its independently computed digest with `sha256`. On success it finalizes the file and sends:

```json
{"type":"file_result","index":0,"status":"verified"}
```

Only after receiving that result may the sender advance to the next manifest index. Before sending a verified result for a non-final file, the receiver advances its current index and creates the next temporary file. If that creation fails, it sends the verified result for the completed file followed immediately by a failed result for the new current index and accepts no DATA for it. Receipt of a verified result for the final index completes the transfer; no additional completion or close message exists.

SHA-256 checks content equality on the paired and separately approved TLS channel. A digest by itself does not authenticate a peer or approve content. TLS remains the sole record-protection layer; the PAKE key is never used as a file-encryption key.

## Failure and cleanup

An early `file_end`, excess DATA, digest mismatch, write error, insufficient storage, or no-replace finalization error fails the current file. The receiver MUST remove its current temporary file, retain the already verified prefix, and send exactly one failed result when the connection is still usable. It MAY send that result while the sender is still sending DATA:

```json
{"type":"file_result","index":0,"status":"failed","code":"hash_mismatch"}
```

No later file is attempted. After a failed result, `cancel`, `error`, timeout, or connection interruption, partial data is deleted and the connection closes. Cleanup failures are handled and logged locally without exposing local paths to the peer.

## Progress

Receiver byte progress is the sum of finalized file sizes plus bytes written for the current file, divided by the checked manifest total. For an all-zero-byte manifest, progress uses verified file count. Sender UI MUST distinguish bytes handed to the transport from files confirmed `verified`; TCP delivery alone is not file completion.

## Deferred

Resume, sparse files, directories, paths, extra metadata, content-addressed chunks, chunk acknowledgements, overwrite, rename, parallel files, and multi-transfer connections are not part of version 1.
