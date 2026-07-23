# Project Overview

## Status

Lanweave is currently a documentation-only project. It describes a future implementation-independent local file-transfer protocol and a Rust CLI reference implementation, but no protocol or application code exists yet.

## MVP Release

The MVP transfers one immutable ordered manifest containing 1..N selected regular files between one sender and one receiver. Each entry contains only a filename and exact size; array position defines order. One successful pairing ceremony admits one connection and one transfer. The receiver reviews the exact whole manifest and prepares its destination before sending an accepting `transfer_response`; preparation failure rejects, and no file bytes are sent before acceptance and `ready`.

Files are sent sequentially. An uninterrupted file uses zero or more raw binary `DATA` frames, followed by `file_end` and at most one `file_result`; interruption may prevent a result. A file becomes part of the verified prefix only after its declared bytes and digest have been checked and its temporary file has been safely finalized.

The transfer stops on the first failure. The receiver deletes the current partial file and keeps the already verified prefix. Later files are not attempted. Whole-manifest atomicity is neither promised nor implied.

Before sending an accepting `transfer_response`, the receiver rejects:

- an empty manifest or a manifest above its fixed bounds;
- path-like or otherwise invalid filename entries;
- duplicate or platform-equivalent destination names;
- names that collide with an existing destination entry; and
- unsafe names or sizes; and
- any destination-preparation failure after local whole-manifest approval.

The manifest cannot be amended after `transfer_request`. There is no second metadata exchange, partial approval, overwrite, or rename negotiation.

## Internal Vertical Slice

The first implementation exercise may constrain the manifest to exactly one file. That one-file path is an internal vertical slice for proving JSON framing, raw `DATA` streaming, verification, temporary-file cleanup, and state transitions.

It is not the MVP release. Release requires the same invariants for multiple sequential files, including a later-file failure that preserves the verified prefix and deletes only the current partial file.

## Protocol Baseline

- Exact experimental protocol version `1`; no version range or capability negotiation.
- TLS 1.3 with a fresh ephemeral identity for each connection; no persistent device identity.
- Receiver-initiated ceremony with a uniform CSPRNG 8-digit decimal code, including leading zeros, privately shared out of band.
- Memory-only, one-use ceremony state with a 120-second monotonic expiry and five failed sender-confirmation verifications total across reconnects; bounded in-flight states and connection/source/CPU rate limits are separate controls.
- Fixed RFC 9382 SPAKE2-P256-SHA256-HKDF-HMAC profile, sender A and receiver B, with no algorithm or profile negotiation. The specified SPAKE2 AAD for confirmation-key derivation contains the 32-byte exporter and exact sender and receiver `hello` JSON body bytes, excluding frame headers; these values are not inserted into RFC transcript `TT`.
- JSON control frames and raw binary `DATA` frames.
- Minimal vocabulary: `hello`, `pairing`, `transfer_request`, `transfer_response`, `ready`, `DATA`, `file_end`, `file_result`, `cancel`, and `error`. Exactly four role- and state-determined `pairing` records carry the two shares and two confirmations.
- Pairing completes before `transfer_request`. The receiver then separately reviews the exact manifest, prepares its destination, and sends an accepting response only if preparation succeeds; pairing never approves unseen files.
- TCP provides ordered reliable delivery; there are no application ACK windows.
- No transfer-complete or close handshake; the final file's verified `file_result` is sufficient protocol completion.
- TLS is the sole record protection. The ceremony is sender-confirmed after the receiver verifies the sender confirmation; mutual pairing additionally requires the receiver confirmation to be flushed to TLS and verified by the sender.

The algorithm and profile are selected, not open research. Release remains blocked on specialist review of the password mapping and exporter invocation and SPAKE2 AAD composition, exact certificate-verifier behavior, deterministic and adversarial vectors, and an independently audited RFC-conformant Rust PAKE implementation and dependency tree. No current crate is approved. An interoperability prototype may include `pakery-spake2` 0.2.1 plus `pakery-core` and `pakery-crypto` with its `p256` and `spake2` features, but the complete surface is very new, unaudited, and candidate-only; returned `Ke`/`session_key` material must be discarded after confirmation. RustCrypto `spake2` is unacceptable because it implements an old draft, is unaudited, and is probably not constant-time. Implementations must not write group arithmetic.

`Provisional TLS` describes only the connection before pairing. A postpairing connection is pairing-authenticated, but remains experimental and must not be treated as safe for sensitive use until the review and audit gates clear.

## Goals

- Direct local transfer without a mandatory service or account.
- Pairing before manifest disclosure, followed by separate exact receiver consent.
- Exact receiver consent for the full manifest.
- Bounded parsing, buffering, file I/O, and queueing.
- Deterministic fail-fast behavior and safe destination handling.
- A small wire format that independent implementations can test.
- mDNS convenience with direct-address fallback.

## Non-Goals

The MVP excludes directories, resume, parallel files or chunks, compression, overwrite or rename negotiation, trusted-device shortcuts, algorithm agility, QUIC, mobile clients, and GUIs. It also does not provide anonymity, protect a compromised endpoint, cross a network that blocks peer connectivity, or make discovery records trustworthy.

## Success Criteria

An MVP candidate must transfer multiple files in manifest order over one pairing-authenticated connection, verify and finalize every successful file, report each result, and satisfy the defined preparation, rejection, failure, and collision behavior on supported platforms. Specialist composition review, an independently audited RFC-conformant PAKE implementation and dependencies, exact profile/interoperability vectors, adversarial tests, fuzzing, and cross-platform filesystem tests are release gates. No sensitive release is permitted before the security gate clears.
