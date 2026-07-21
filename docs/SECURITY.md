# Security

> **Experimental:** Lanweave has not been audited. Until the protocol and implementation receive specialist review, early builds should be treated as development software and should not be used for sensitive transfers.

## Objectives

| Objective | Required property |
| --- | --- |
| Confidentiality | File metadata/content and secrets are unreadable to network observers after channel protection; discovery leaks are minimized |
| Integrity | AEAD detects message changes; independent SHA-256 verifies finalized file bytes |
| Peer authentication | Secure session binds both device identities, human pairing, key exchange, roles, and transcript |
| Explicit consent | Receiver locally approves summary before token creation and any file data |
| Replay resistance | Fresh IDs/challenges/ephemeral keys, transcript binding, per-session sequences, expiry and replay caches |
| Token-guessing resistance | CSPRNG token, short lifetime, ≤5 attempts, pacing/backoff, preferably reviewed PAKE/proof |
| Path safety | Locally selected root, normalized single components, no absolute/traversal paths, safe no-follow writes |
| Malicious metadata defense | Schema bounds, integer checks, sanitization, no terminal escape rendering |
| Bounded resources | Frame/count/queue/concurrency/time/storage limits before expensive operations |
| Safe failure | No finalization on uncertainty; erase secrets, stop I/O, clean/quarantine partials, expose coarse remote errors |

## Boundaries and assumptions

The things I want Lanweave to protect are the selected file content and metadata, pairing tokens, identity private keys, session secrets, the receiver's filesystem, and the receiver's decision to accept a transfer. Data crosses a trust boundary when it arrives through mDNS or a socket, enters a parser, reaches cryptographic verification, becomes a filename, is written to storage, or appears in a log or prompt.

I assume both endpoints are honest at the time of transfer, the operating system provides secure randomness, the selected cryptographic libraries behave as documented, and the user reads the token from the intended receiver. Lanweave cannot protect an already compromised machine, an unlocked stolen account, a user who opens a malicious received file, or a user who approves the wrong physical device. It also does not hide traffic volume and timing or prevent network jamming.

## Before and after session confirmation

Discovery is unauthenticated. `DeviceHello`, negotiation, the transfer summary, the receiver's decision, token messages, and the logical key-exchange messages all happen before the **pairing-authenticated secure session** exists.

The proposed TCP profile starts TLS early so those messages are hidden from passive observers where possible. They are still untrusted until pairing succeeds because an active peer or MITM may control the other end of that provisional channel.

I deliberately keep pre-session behavior small: no full local paths, file content, destination paths, private keys, or session secrets; no compression; no side effects beyond pending state; and only bounded control frames under short deadlines and rate limits. `FileMetadata`, `TransferReady`, chunks, acknowledgements, completion, and the normal close flow require the authenticated secure session. Before then, an error or cancellation may only carry coarse details for its current state.

### Complete message protection map

| Minimum class | Messages | Additional requirement |
| --- | --- | --- |
| P0/P1 | `DeviceHello`, `DeviceGoodbye` | Claims are untrusted; goodbye is advisory. Use P2 if a secure session already exists. |
| P1 | `ProtocolNegotiation`, `ProtocolNegotiationAccepted`, `ProtocolNegotiationRejected` | Exact offers/selection are later transcript-bound. |
| P1 | `TransferRequest`, `TransferAccepted`, `TransferRejected` | Bounded summary and explicit local decision; no file content. |
| P1 confidential | `PairingTokenChallenge`, `PairingTokenSubmission`, `PairingTokenAccepted`, `PairingTokenRejected` | Challenge has no token; direct submission forbidden without channel confidentiality; result is not session authentication. |
| Reviewed handshake | `KeyExchangeInit`, `KeyExchangeResponse` | Profile data, identities, approval, token result, and negotiation are bound; failure closes. |
| P2 confirmation | `SessionEstablished` | Protected with handshake-derived confirmation context; file messages wait for both valid confirmations. |
| P2 | `FileMetadata`, `TransferReady`, `FileChunk`, `ChunkAcknowledgement`, `FileComplete`, `TransferComplete`, `SessionClosed` | Encrypted, authenticated, sequenced, ID/state validated. |
| P1 or P2 by active state | `TransferCancelled`, `Ping`, `Pong`, `Error` | Cannot downgrade an active P2 context; errors are coarse and never loop. |

## Pairing tokens

- Generate with the OS CSPRNG; choose entropy/alphabet based on usability testing and threat analysis.
- Bind to token ID, pairing-session ID, request/transfer, receiver approval, both instances, and transcript.
- Expire after 60 seconds; maximum five attempts; no deadline extension after failures.
- Consume on success, expiry, cancellation, attempt exhaustion, process restart, or session failure.
- Store only in memory or verifier form; do not persist, echo in sender UI after submission, place in argv, or log.
- Compare verifier/proof values in constant time where meaningful.
- Return generic errors and apply cross-connection source/device backoff.
- Prefer a reviewed PAKE/proof; plaintext submission is experimental even over provisional TLS.

## Filesystem safety

- Version 1.0 accepts regular files only. Reject directories, devices, FIFOs, sockets, and symlinks at source and destination boundaries.
- Treat received names as display suggestions. Remove/replace separators and controls, reject `.`/`..`, absolute/prefix paths, NUL, reserved device names, trailing-dot/space ambiguities, and names illegal on the destination.
- Open a receiver-selected directory handle and create unpredictable temporary files with create-new/no-follow semantics where supported. Revalidate containment and object type.
- Never follow a remote-provided directory hierarchy. Deep paths are unsupported initially.
- Default on collision is “ask/rename/reject,” never silent overwrite. If overwrite is approved, replace atomically only after hash verification and preserve failure recovery where possible.
- Reserve/check space before `TransferReady`, enforce actual byte limits while writing, and handle sparse-file/accounting differences.
- Hash while writing; close/flush as policy requires; finalize by atomic same-filesystem rename. Cross-filesystem moves need a separate safe flow.
- Partial files use non-user-facing names and are deleted by default on failure; inability to delete is reported locally without exposing paths remotely.
- Do not automatically execute, preview, or open received files.

## Network and parser safety

- Validate header magic/version/flags and length before allocation; cap control/data separately.
- Reject duplicate critical fields, invalid UTF-8 where required, non-canonical encodings where transcript/signature semantics require them, integer overflow, invalid enum values, and trailing bytes.
- Bound pending connections, requests, pairing sessions, queued writes, chunks in flight, and per-source work.
- Use staged timeouts and cheap validation before signature/hash/disk operations.
- Do not add compression initially. If added, negotiate it after authentication and cap decompressed bytes/ratio/time.
- Cryptographic failure, nonce/sequence violation, or contradictory identity closes the connection without diagnostic detail.

## Logs and UI

Use structured event codes and locally generated safe text. Redact tokens, paths (unless opt-in), stable IDs/fingerprints, peer-provided error strings, keys, transcript material, and file content. Escape or strip terminal control characters from all network names. Debug logging must not weaken these rules. Remote errors expose only codes and bounded safe text.

## Operational guidance

Bind only intended interfaces, show firewall implications, display authenticated identity/fingerprint after pairing, and warn on identity changes for any future trusted-device record. Rate-limiting by IP alone is insufficient on IPv6/shared hosts; combine endpoint, instance/device claims (untrusted), global budgets, and proof-of-work only if later justified.

See [Threat Model](THREAT_MODEL.md), [Cryptography](CRYPTOGRAPHY.md), and [File Transfer](FILE_TRANSFER.md).
