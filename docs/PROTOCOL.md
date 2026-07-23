# Lanweave Protocol Specification

**Wire protocol:** experimental version 1

**Status:** Draft; selected pairing profile and implementation review remain release-blocking

This document is normative. The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** describe interoperability requirements.

## Scope

Version 1 transfers an immutable ordered manifest of regular files between one sender and one receiver over one framed TCP/TLS connection. One connection carries at most one transfer request and its transfer. Directories, paths, symlinks, special files, multiple transfers per connection, and internet routing are outside the protocol.

The LAN, discovery data, remote input, and peer identity before pairing are untrusted. Pairing occurs before the manifest so filenames and sizes are not revealed to an unauthenticated endpoint. The selected experimental profile is specified in [Cryptography](CRYPTOGRAPHY.md), but specialist review and an approved RFC-conformant implementation remain release-blocking.

## Fixed profile

Protocol version 1 has no Lanweave, application, or PAKE negotiation. The peers use TLS 1.3, ALPN `lanweave/1`, RFC 9382 SPAKE2-P256-SHA256-HKDF-HMAC, the fixed framing and messages in [Message Format](MESSAGE_FORMAT.md), and SHA-256 file digests. Standard TLS 1.3 cipher-suite and key-exchange negotiation MAY occur only under one pinned, reviewed `rustls` cryptographic-provider policy. A peer MUST configure only TLS 1.3 and MUST NOT offer, select, or fall back to another protocol version, Lanweave version, provider policy, pairing method, or application algorithm on the same connection.

The receiver creates one memory-only, uniformly random 8-character decimal code with a 120-second monotonic lifetime. It preserves leading zeroes, permits five failed admitted sender-confirmation verifications total across connections, and enforces expiry, verification admission, one-use consumption, and attempt accounting atomically under concurrency. The code is shared privately out of band and never sent on the wire.

The sender is SPAKE2 Party A and the receiver is Party B. Pairing authenticates and binds the exact `hello` frame bodies to the current TLS exporter. It does not bind the later, as-yet unseen manifest or approval. TLS remains the sole application record layer; no application AEAD is added.

## Connection sequence

Messages and frame encoding are defined in [Message Format](MESSAGE_FORMAT.md).

1. The peers establish one TCP connection and complete TLS 1.3, including Finished verification. The receiver sends exactly one well-formed DER self-signed certificate, no intermediates, with an ECDSA P-256 SPKI; its self-signature verifies with that SPKI as a shape check, not a trust check. Its only permitted `CertificateVerify` scheme is `ECDSA_NISTP256_SHA256`. The sender's provisional verifier intentionally does not use chain trust, name, SAN, validity, or EKU as identity checks, but it MUST reject TLS 1.2 verification and verify the TLS 1.3 signature through the pinned `rustls` provider helper. Both peers then check exact TLS 1.3 and ALPN `lanweave/1`. TLS 1.2, 0-RTT, resumption, tickets, and pre-handshake or half-RTT application data are forbidden.
2. After TLS completes, the sender sends exactly one `hello`. The receiver validates and retains its exact JSON body bytes, then sends exactly one `hello` response and retains its exact bytes. Each exact body is at most 4,000 bytes and contains `version: 1`. An otherwise valid different non-negative integer version causes terminal `unsupported_version`; a missing or wrongly typed version or an oversized `hello` is `invalid_message`.
3. The sender sends one Party A `pairing` share. The receiver sends one Party B share. Each share is a canonical unpadded-base64url encoding of a 65-byte uncompressed P-256 point.
4. The sender sends one 32-byte HMAC-SHA256 confirmation in a `pairing` record. If and only if it verifies, the receiver atomically consumes the code and sends its confirmation. The sender verifies that confirmation. Both directions require RFC 9382 key confirmation; the receiver never confirms first.
5. Each peer marks pairing successful only after its outgoing framed confirmation has been accepted by the TLS writer and successfully flushed, and the applicable peer confirmation has verified. This commitment does not prove peer delivery. A lost receiver confirmation therefore fails the sender safely while leaving the code consumed. SPAKE2 secret material receives best-effort zeroization and is not used to encrypt application data.
6. Only after mutual pairing success, the sender sends exactly one `transfer_request`. Its `files` array is the complete immutable manifest; array position is file order.
7. The receiver validates the entire request before asking for or recording consent. It derives count and total size, validates limits and names, chooses the destination, and rejects duplicate, destination-platform-equivalent, or already-existing destination names. It separately presents the complete manifest to the user for approval.
8. After approval, the receiver prepares the destination and required resources before accepting. Preparation failure sends a rejecting `transfer_response` with the applicable existing reason. Only successful preparation permits an accepting `transfer_response`, which covers every entry without modification.
9. Immediately after the accepting `transfer_response`, the receiver sends `ready`. There is no accepted-but-not-ready application failure state; a transport write failure still terminates the connection. Rejection terminates the transfer, and no subset, rename, or overwrite response exists. The sender MUST NOT send DATA before `ready`. `ready` relies on the authenticated TLS connection and records transfer readiness; it is not a PAKE confirmation.
10. For each manifest entry in order, the sender sends zero or more DATA frames totaling exactly the declared size, followed by `file_end(index, sha256)`. DATA implicitly belongs to the current index.
11. The receiver verifies the exact byte count and SHA-256, finalizes the file without overwrite, and sends `file_result(index, status="verified")`.
12. The sender MUST wait for that verified result before sending DATA for the next file.
13. The final verified `file_result` completes the transfer. Either peer MAY then close the connection; there is no `TransferComplete`, acknowledgement of completion, or close handshake.

## Pairing state and attempts

Pairing has exactly four records and no retry within a connection: sender share, receiver share, sender confirmation, receiver confirmation. Record direction and state determine the SPAKE2 role. Any extra, duplicate, reversed, or wrong-state pairing record is terminal.

A failed attempt is charged only when a syntactically valid and correctly ordered sender confirmation is admitted to verification against an active ceremony and fails. It is not a count of connections, shares, PAKE computations, malformed JSON, invalid encodings or points, wrong-state messages, or disconnects. Those events do not decrement the code budget. They MUST still be bounded by handshake deadlines, finite global, per-source, and per-ceremony in-flight pairing-state limits, and connection and CPU rate limits so they cannot trigger unlimited expensive work.

The five-attempt budget spans concurrent and successive connections for the same code ceremony. Verification admission and failed-attempt increments MUST be atomic; concurrent attempts MUST NOT create a sixth admitted verification opportunity. Wrong-code, exporter-binding, `hello`-binding, and otherwise valid sender-confirmation failures share only `authentication_failed` when safe, or a silent close. A correctly encoded and ordered receiver confirmation that fails sender-side verification has the same terminal outcome but does not affect the receiver's counter. An unavailable, expired, exhausted, consumed, or nonexistent ceremony MAY fail earlier at any pairing phase with the same code or a silent close. No dummy PAKE is required or permitted merely to hide ceremony state. Phase, timing, resource use, and ceremony availability are not hidden, and existence or expiry is not promised to be indistinguishable. No response reports attempts remaining.

## Admission rules

A request MUST contain 1 through 1,024 selected regular files. Every manifest entry contains only a filename and exact size. The receiver MUST reject the whole request before acceptance if:

- the request or any field exceeds the fixed limits;
- a name is invalid on the destination platform or represents a path;
- two names are equal under destination-platform comparison rules;
- any final destination already exists;
- checked file-count, size, storage, or resource policy fails;
- the user rejects the complete manifest.

The receiver MUST NOT accept a subset, alter order, rename a file, or overwrite an existing destination. It MUST use no-replace finalization so a destination created after preflight causes a file failure rather than an overwrite. The sender is responsible for ensuring each source is a regular file.

## Streaming rules

Files are transferred sequentially. DATA has no transfer ID, file ID, index, ordinal, offset, or application flow-control field. The receiver tracks the current manifest index and accepted byte count from protocol state.

Each DATA frame MUST be at most 1 MiB and MUST NOT exceed the current file's remaining declared size. The sender sends no DATA for a zero-byte file. There are no chunk acknowledgements, application receive windows, resume points, or parallel files. Implementations MUST use TCP backpressure and bounded read, write, and queue buffers rather than buffering a whole file.

After exactly the declared bytes, the sender sends `file_end` with the current index and lowercase SHA-256. The receiver independently hashes bytes as received. A verified file is finalized before its successful `file_result` is sent. If another manifest entry remains, the receiver prepares that next temporary file before sending the current verified result. If preparation fails, it sends the current verified result followed immediately by a failed result for the new current index; no DATA for that index is accepted.

## Failure and termination

Pairing authentication failure is terminal and follows the coarse confirmation-failure behavior above. Secret cleanup is best-effort zeroization, not guaranteed erasure. On ceremony consumption, the receiver MUST cancel all other active pairing connections for that ceremony. Each owner promptly best-effort zeroizes its owned code, code digest, `w`, local `x` or `y`, `K`, secret transcript intermediates, `Ke`, `Ka`, confirmation keys, and expected tags as soon as no longer needed, including on success, failure, timeout, disconnect, expiry, exhaustion, or cancellation. Ephemeral `rcgen` and `rustls` private-key-buffer erasure remains a release-review gate.

On the first file failure, the receiver MUST stop the transfer, remove the current temporary file, retain any already verified prefix of the manifest, and send `file_result` with the current index, `status: "failed"`, and a failure code when the connection is usable. It MAY send this result while the sender is still sending DATA. Later files MUST NOT be attempted. The connection then closes.

An early `file_end`, excess DATA for the active file, hash mismatch, write failure, or no-replace finalization failure is a file failure reported with failed `file_result`. DATA in the wrong state, a control message in the wrong state, malformed input, a connection-level resource violation, or version mismatch is a terminal `error` when safe.

Failure response precedence is fixed. Unsafe framing or the generic hard frame bounds close without a response. Safely framed malformed JSON, wrong JSON types, unknown fields, unknown messages, wrong-state controls, invalid control indices, and oversized `hello` or `pairing` messages use `invalid_message`, except the pairing confirmation failures described above use only `authentication_failed` or silent close. A structurally valid `transfer_request` over 256 KiB, or a safely parsed request that otherwise fails semantic manifest admission, uses a rejecting `transfer_response` with reason `invalid_manifest`. Active-file size, hash, write, storage, and finalization failures use failed `file_result`.

`cancel` is terminal in every non-terminal state. On cancellation, interruption, or timeout, each peer stops work; the receiver removes the current temporary file and releases resources for unattempted entries while retaining verified files. Partial files MUST NOT appear under final names. Version 1 has no resume.

After sending or receiving terminal `cancel`, `error`, or failed `file_result`, a peer MUST NOT process further frames and MUST close the connection. Transport EOF is failure unless the final verified `file_result` has completed the transfer.

## Fixed limits

| Item | Version 1 limit |
| --- | ---: |
| Filename | 255 UTF-8 bytes |
| Files in manifest | 1 to 1,024 |
| `transfer_request` / manifest frame | 256 KiB |
| Exact `hello` JSON body | 4,000 bytes |
| `pairing` JSON body | 4,096 bytes |
| Generic JSON allocation bound | 1 MiB |
| DATA frame | 1 MiB |
| Integer, file size, checked total size | `2^53-1` |
| Pairing code lifetime | 120 monotonic seconds |
| Failed admitted sender-confirmation verifications per code | 5 total across connections |

Limits are not negotiated. The generic 1 MiB JSON body limit is the hard pre-allocation bound. The message-specific 4,000-byte `hello`, 4,096-byte `pairing`, and 256 KiB `transfer_request` limits apply only after a bounded parse identifies the message type. Implementations MUST NOT allocate a claimed JSON body over 1 MiB. A receiver MAY apply a lower local storage or resource policy by rejecting the whole request, but MUST parse and report that rejection within the fixed wire bounds.

## Deferred

Multiple transfers per connection, directories and paths, metadata beyond name and size, subset acceptance, overwrite or rename policies, resume, chunk acknowledgements, application flow windows, parallelism, capability negotiation, algorithm negotiation, resumption, and trusted-device pairing are deferred beyond version 1.
