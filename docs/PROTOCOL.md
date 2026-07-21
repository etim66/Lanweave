# Lanweave Protocol Specification

**Specification revision:** 0.1  
**Proposed wire protocol:** 1.0  
**Status:** Draft; security-sensitive portions require review

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** express interoperability requirements in this draft. Proposed values remain subject to research and review.

## Purpose

The Lanweave Protocol coordinates discovery hints, consent, pairing, authenticated secure-session establishment, and bounded transfer of regular files between two LAN peers. It defines messages, states, validation, version negotiation, and failure behavior independent of UI and programming language.

It does not route across the internet, guarantee discovery, synchronize folders, define a cloud service, authenticate a person's legal identity, make a compromised endpoint safe, or let remote paths control the local filesystem.

## Roles and entities

Canonical definitions of local device, peer, sender, receiver, initiator, responder, transfer request, pairing session, secure session, transfer, file, chunk, device identity, instance identity, and session identity are in the [Glossary](GLOSSARY.md). Sender/receiver describe data direction; initiator/responder describe connection direction and MUST NOT be assumed to be permanent authority roles.

## Design assumptions

- The LAN and discovery channel are untrusted.
- Receiver interaction is available for each initial transfer.
- Local monotonic clocks enforce deadlines; remote wall-clock timestamps are advisory.
- Only regular files are supported in 1.0.
- Full `FileMetadata` and all file bytes occur only after session confirmation.
- A transport may provide encryption before pairing, but the protocol does not call it a secure session until pairing, identities, version, key exchange, and transcript are authenticated together.

## Phases and allowed messages

| # | Phase | Principal messages | Protection requirement |
| ---: | --- | --- | --- |
| 1 | Discovery | mDNS advertisement; optionally `DeviceHello`, `DeviceGoodbye` after connect | Advertisement is unauthenticated; no secrets |
| 2 | Connection establishment | transport handshake | Authenticate bounds/endpoints where possible; no file data |
| 3 | Protocol negotiation | `ProtocolNegotiation`, `ProtocolNegotiationAccepted`, `ProtocolNegotiationRejected` | Downgrade-resistant transcript binding required before session confirmation |
| 4 | Transfer request | `TransferRequest` | Summary metadata only; bounded and rate-limited |
| 5 | Receiver decision | `TransferAccepted`, `TransferRejected` | Explicit local decision; response correlated to request |
| 6 | Pairing-token challenge | `PairingTokenChallenge` | Never contains token; receiver displays token locally |
| 7 | Token submission | `PairingTokenSubmission`, `PairingTokenAccepted`, `PairingTokenRejected` | Confidential channel strongly preferred; attempts bound to challenge and rate-limited |
| 8 | Secure key establishment | `KeyExchangeInit`, `KeyExchangeResponse` | Reviewed construction; bind identities, ephemeral keys, token result, version, and transcript |
| 9 | Session confirmation | `SessionEstablished` in both directions | First point at which pairing-authenticated secure session is usable |
| 10 | File metadata | `FileMetadata`, `TransferReady` | MUST be encrypted and authenticated |
| 11 | File data | `FileChunk`, `ChunkAcknowledgement` | MUST be encrypted/authenticated, ordered, bounded, and flow-controlled |
| 12 | Integrity verification | `FileComplete` | Receiver independently verifies length and SHA-256 |
| 13 | Completion/termination | `TransferComplete`, `TransferCancelled`, `Error`, `SessionClosed`; `Ping`/`Pong` while active | Protected when session exists; failures close safely |

`Error`, `TransferCancelled`, `Ping`, and `Pong` are state-dependent cross-cutting messages. The precise pre-authentication set and secure flags appear in [Message Format](MESSAGE_FORMAT.md).

## Successful transfer

1. The receiver advertises `_lanweave._tcp.local.` with minimal, untrusted discovery metadata.
2. The sender resolves candidate addresses and connects using the selected transport profile.
3. Peers exchange `DeviceHello` and negotiate a common major/minor version, required capabilities, framing version, and algorithms. The chosen parameters are recorded in the transcript.
4. The sender creates a unique request ID and sends `TransferRequest` with file count, total bytes, and sanitized summaries, never file content or sender-local absolute paths.
5. The receiver validates bounds, checks policy/preliminary storage, presents the request, and explicitly accepts.
6. The receiver creates a transfer ID, pairing-session ID, token ID, random one-time token, deadline, and attempt budget. It displays the token locally and sends `TransferAccepted` then `PairingTokenChallenge` without the token.
7. The sender submits the user-entered value. The receiver performs constant-time verification where applicable, decrements the attempt budget, consumes the token on success, and sends only a generic result.
8. Peers execute the reviewed authenticated key-establishment profile. The construction MUST bind both device identities, instance IDs, request/transfer/pairing IDs, token-challenge result, negotiated version/capabilities, ephemeral contributions, and the transcript. The exact construction is a security-review gate; [Cryptography](CRYPTOGRAPHY.md) does not claim a novel protocol is secure.
9. Each peer verifies the binding and sends protected `SessionEstablished` confirmation containing the same secure-session identity/transcript result. File messages remain forbidden until both confirmations validate.
10. The sender sends one `FileMetadata` per file. The receiver validates names, counts, lengths, hashes, destination policy, and available space, prepares temporary destinations, and sends `TransferReady`.
11. The sender streams bounded `FileChunk` messages under negotiated flow control. Receiver writes at declared offsets, hashes accepted bytes, and returns cumulative/checkpoint `ChunkAcknowledgement` messages.
12. After each complete byte stream, sender sends `FileComplete` with declared final length/hash. Receiver flushes as required, compares its independent length/hash, and atomically renames the temporary file if policy permits.
13. When all files verify, receiver sends `TransferComplete`; sender confirms with `TransferComplete` using a distinct message ID and correlated transfer ID.
14. Either peer initiates `SessionClosed`; the other acknowledges, both erase ephemeral/session secrets, and the transport closes within the grace period.

## Failure behavior

| Failure | Required behavior |
| --- | --- |
| Receiver rejection | Send `TransferRejected` with coarse reason; terminate request and consume no token. Connection MAY remain for another request subject to policy. |
| Expired request | Reject with `ExpiredRequest`; delete pending state and ignore late acceptance/submission. |
| Incorrect token | Send `PairingTokenRejected(InvalidToken)` without revealing closeness; decrement attempts; remain waiting if attempts/deadline remain. |
| Expired token | Consume it, send `ExpiredToken`, fail pairing, and require a new receiver decision/challenge. |
| Too many attempts | Consume token, rate-limit source/device context, send `TooManyTokenAttempts`, and close the pairing/connection. |
| Unsupported version | Send `ProtocolNegotiationRejected` with local supported range but no transfer processing; close gracefully. |
| Connection interruption | Mark session/transfer failed, close temporary handles, retain or delete partials per explicit policy; initial version does not resume. |
| Insufficient storage | Before data, reject readiness with `InsufficientStorage`; during data, abort transfer, never finalize partial files. |
| Invalid message | Send a safe `Error` only if framing/state permits; repeated, ambiguous, or cryptographic failures close immediately. |
| Cryptographic verification failure | Send at most generic `AuthenticationFailed` if safe, erase secrets, fail session, and close. Do not retry inside the same transcript. |
| File-hash mismatch | Do not finalize; send `HashMismatch`, quarantine/delete partial per local policy, fail file and initially the entire transfer. |
| User cancellation | Send protected `TransferCancelled` when possible, stop new chunks, clean partials, and reach `Cancelled`. |
| Peer disappearance | Expire discovery entry; active connection state, not mDNS, determines active transfer failure. |
| Session timeout | Send `SessionTimeout` if safe, erase session state, clean partials, and close. |

## Invariants

1. File data MUST NOT be sent before explicit receiver approval.
2. `FileMetadata` and file data MUST NOT be sent before both peers confirm the secure session.
3. Pairing tokens MUST be unpredictable within the chosen human format, short-lived, single-use, bound to one receiver decision and pairing session, and rate-limited.
4. A token MUST NOT be used directly as an encryption key or long-term key.
5. Each request, token challenge, pairing session, secure session, transfer, file, message, device installation, and process instance uses the correct distinct identifier namespace.
6. A message is accepted only in a valid state, from the expected role, with valid correlation and bounds.
7. Encrypted messages MUST use authenticated encryption and replay-resistant sequence/nonces.
8. A completed file MUST match declared length and SHA-256 computed from received bytes before finalization.
9. Remote names/paths MUST NOT select arbitrary local destinations, escape the receiver root, or cause silent overwrite.
10. The receiver MUST explicitly approve each transfer unless a future separately negotiated trusted-device policy specifies otherwise.
11. Negotiation MUST finish before any transfer-specific message is processed and MUST be transcript-bound against downgrade.
12. Discovery claims MUST be treated as untrusted until the secure session binds identity.
13. Untrusted lengths/counts MUST be checked before allocation, hashing, decompression, or disk reservation.
14. Unknown critical fields/messages MUST fail; unknown optional fields MAY be ignored only under the rules in [Versioning](VERSIONING.md).

## Recommended timeouts

Values are implementation defaults, not immutable wire requirements unless negotiated. A peer MAY enforce stricter local policy and should expose human-action deadlines to the UI.

| Timer | Default | Starts / resets | Expiry action |
| --- | ---: | --- | --- |
| Discovery expiry | 120 s or 2× advertised TTL | Valid advertisement | Remove from browse results; never kill active connection |
| Connection timeout | 10 s per candidate; 20 s overall | Connect attempt | Try next resolved address, then fail |
| Transfer-request timeout | 120 s | Valid request received | `ExpiredRequest`; delete pending request |
| Pairing-token timeout | 60 s | Challenge creation; never extended by failure | Consume token; fail pairing |
| Token attempts | 5 total; at least 1 s between attempts | Challenge creation | Fifth failure consumes token and closes pairing |
| Secure-session idle | 5 min | Any valid protected control/data message | Close session; active UI may negotiate up to 30 min |
| Transfer inactivity | 30 s | Accepted data/progress/flow-control activity | Ping once, then fail after an additional 10 s |
| Graceful close | 5 s | `SessionClosed` sent | Close transport and erase secrets |

Local backoff SHOULD rate-limit repeated requests across connections (for example, exponential delay capped at 15 minutes). Exact anti-DoS policy remains implementation-specific.

## Recommended limits

| Item | Proposed 1.0 wire maximum | Implementation policy |
| --- | ---: | --- |
| Device display name | 128 UTF-8 bytes | MAY display/truncate fewer graphemes |
| Filename component | 255 UTF-8 bytes on wire | MUST also obey destination OS rules; opaque-name extension deferred |
| Files per request | 1,024 | Default SHOULD be 100; receiver may reject lower |
| Request metadata | 256 KiB | Counts toward 1 MiB control-frame limit |
| Control message/frame | 1 MiB | Allocate only after header validation |
| File chunk payload | 1 MiB maximum | Negotiate 256 KiB default; permit 64 KiB–1 MiB |
| Data frame | 1,048,640 bytes | Chunk plus bounded envelope overhead |
| Total transfer size | Unsigned 64-bit semantic field | Receiver policy/storage decides; no unconditional allocation |
| Remote error text | 512 UTF-8 bytes | Prefer code and local generic rendering |
| Pending incoming requests | No universal wire value | Default 8 per instance and 2 per source/device ID |
| Token attempts | Maximum 5 per challenge | Implementations MUST NOT raise without security review |

Wire maxima protect interoperability and parsers; lower file-count, size, concurrency, storage, and display limits are local policy and must yield structured rejection rather than undefined behavior. Integer arithmetic MUST detect overflow.

## Open security gates

- Select a reviewed way to bind the low-entropy token to TLS/device identity (PAKE, TLS exporter plus proof, or another analyzed construction).
- Decide whether the first transport starts TLS immediately or performs a tightly limited plaintext preface; plaintext token submission is not recommended for a security-stable release.
- Specify certificate/raw-public-key validation and device key-change UX.
- Obtain review of transcript encoding, replay cache scope, AEAD nonce construction, and key separation.

See [Message Format](MESSAGE_FORMAT.md), [State Machines](STATE_MACHINES.md), and [Security](SECURITY.md).
