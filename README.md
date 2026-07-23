# Lanweave

Lanweave is a planned local peer-to-peer file transfer protocol and Rust CLI. This repository currently contains documentation only: there is no working discovery, networking, security, CLI, or transfer implementation yet.

The first release is intentionally narrow. It will transfer an explicitly approved, ordered manifest of regular files over one TCP/TLS connection, using JSON control frames and raw binary `DATA` frames.

## Intended Flow

1. The receiver begins a pairing ceremony, generates a uniform CSPRNG 8-digit decimal code, and shares it privately out of band. The leading-zero-preserving code is memory-only, one-use, expires after 120 seconds on a monotonic clock, and permits five failed sender-confirmation verifications total across reconnects. Bounded in-flight pairing states and separate connection, source, and CPU rate limits constrain work that never reaches that verification.
2. The sender finds the receiver through mDNS/DNS-SD or supplies its address directly. The peers open TLS 1.3 using an ephemeral identity generated for that connection, then exchange `hello`. Experimental protocol version `1` and the security profile are fixed rather than negotiated.
3. The peers run RFC 9382 SPAKE2-P256-SHA256-HKDF-HMAC, with the sender as A and receiver as B. Four `pairing` records carry the sender share, receiver share, sender confirmation, and receiver confirmation. The specified SPAKE2 AAD for confirmation-key derivation contains the 32-byte TLS exporter and both exact `hello` JSON body byte strings, excluding frame headers; these values are not inserted into RFC 9382 transcript `TT`. The ceremony becomes sender-confirmed when the receiver verifies the sender confirmation, while mutual pairing requires the receiver confirmation to be flushed to TLS and verified by the sender.
4. Only after pairing succeeds, the sender sends one `transfer_request` containing the complete immutable manifest of 1..N selected regular files.
5. The receiver separately reviews that exact whole manifest and prepares the destination before sending `transfer_response`. Preparation failure rejects the request; an accepting response is sent only after preparation succeeds. Pairing never approves unseen files. There is no later metadata phase and no partial-manifest approval.
6. After the accepted response, the receiver sends `ready`, then files are sent sequentially as raw binary `DATA` frames. An uninterrupted file is followed by `file_end` and at most one `file_result`.
7. A successful file is verified and finalized before the next file starts. On the first failure, the current partial file is deleted, the already verified prefix is kept, and the transfer stops.

Duplicate names, platform-equivalent names, and names that already exist in the destination are rejected. Version 1 does not negotiate overwrite or rename behavior.

## MVP Boundary

The MVP release requires multi-file transfer. A one-file path is only an internal vertical slice used to prove the framing, storage, and state-machine path; it is not a releasable MVP by itself.

The version 1 message vocabulary is deliberately small:

- JSON control frames: `hello`, `pairing`, `transfer_request`, `transfer_response`, `ready`, `file_end`, `file_result`, `cancel`, and `error`.
- Binary frame: `DATA`, whose payload is uninterpreted file bytes.

There is no CBOR, capability negotiation, second metadata phase, application ACK window, transfer-complete message, or close handshake. TCP supplies reliable ordered delivery. The selected pairing profile uses the live connection's exporter and exact `hello` JSON bodies in SPAKE2 AAD for confirmation-key derivation, not in `TT`; TLS remains the sole record-protection layer.

## Deferred Scope

Directories, resume, parallelism, compression, overwrite or rename negotiation, trusted devices, algorithm agility, QUIC, mobile clients, and graphical interfaces are outside the MVP.

## Initial Implementation Shape

The initial Rust implementation will be one binary crate with internal modules for the CLI, orchestration, protocol/state validation, JSON and frame parsing, transport, pairing, discovery, and filesystem transfer. It should use a small pairing-library adapter and a ceremony manager rather than implement group arithmetic. Preferred dependency families are `rustls` 0.23, `tokio-rustls` 0.26, `rcgen` 0.14, and `zeroize`. An interoperability prototype may also expose the full candidate surface of `pakery-spake2`, `pakery-core`, and `pakery-crypto` with its `p256` and `spake2` features; this surface is candidate-only, and returned `Ke`/`session_key` material must be discarded after confirmation. Crates will be split only after a concrete reuse or isolation need appears.

## Documentation

| Document | Purpose |
| --- | --- |
| [Project overview](docs/PROJECT_OVERVIEW.md) | MVP scope, internal vertical slice, invariants, and non-goals |
| [Protocol](docs/PROTOCOL.md) | Normative version 1 sequence, admission rules, limits, and failure behavior |
| [Message format](docs/MESSAGE_FORMAT.md) | Strict JSON messages, raw `DATA`, and fixed stream framing |
| [State machines](docs/STATE_MACHINES.md) | Sender and receiver states and shared invariants |
| [Sequence diagrams](docs/SEQUENCE_DIAGRAMS.md) | Successful, rejected, failed, and cancelled exchanges |
| [File transfer](docs/FILE_TRANSFER.md) | Manifest, destination, streaming, verification, and cleanup rules |
| [Transport](docs/TRANSPORT.md) | Single-connection TCP/TLS behavior and backpressure |
| [Cryptography](docs/CRYPTOGRAPHY.md) | Selected pairing profile and TLS-binding requirements |
| [Security](docs/SECURITY.md) | Security objectives, boundaries, and implementation requirements |
| [Threat model](docs/THREAT_MODEL.md) | Threats, mitigations, residual risks, and review gates |
| [Error handling](docs/ERROR_HANDLING.md) | Terminal errors, file failures, cancellation, and cleanup |
| [Versioning](docs/VERSIONING.md) | Exact version 1 and strict schema compatibility |
| [Design decisions](docs/DESIGN_DECISIONS.md) | Compact accepted, research, and deferred decision log |
| [Implementation roadmap](docs/IMPLEMENTATION_ROADMAP.md) | Dependency gates from framing through release |
| [Architecture](docs/ARCHITECTURE.md) | One-crate module boundaries, data flow, and concurrency rules |
| [Research plan](docs/RESEARCH_PLAN.md) | Security review and focused feasibility prototypes |
| [Testing strategy](docs/TESTING_STRATEGY.md) | Wire, state, filesystem, fuzz, and security-profile tests |
| [Discovery](docs/DISCOVERY.md) | Minimal untrusted mDNS hints and direct-address fallback |
| [Glossary](docs/GLOSSARY.md) | Canonical version 1 terminology |

## Development Status

The project is in protocol consolidation and research. Work should proceed by dependency gates: wire/framing, local multi-file transfer, TCP/TLS transport, audited pairing implementation, discovery, then hardening and release. No implementation milestone has been completed.

## Security Status

Lanweave has not been implemented, audited, or shown safe for sensitive data. `Provisional TLS` names only the prepairing connection. After mutual confirmation the connection is pairing-authenticated, but the protocol and implementation remain experimental and unsafe for sensitive use until audit.

Release blockers include specialist review of the application-specific password-to-scalar mapping; audit of the complete implementation dependency tree, including the candidate `pakery-spake2`/`pakery-core`/`pakery-crypto` surface; review of the exporter invocation and SPAKE2 AAD composition; exact `rustls` certificate-verifier behavior; and deterministic RFC, profile, interoperability, and adversarial vectors. No current PAKE crate is approved: `pakery-spake2` 0.2.1 is a very new, unaudited interoperability-prototype candidate only, while RustCrypto `spake2` targets an old draft, is unaudited, and is probably not constant-time. No sensitive release may proceed until every gate clears.

## Contributing And Licence

Design feedback is welcome, especially when it identifies a violated invariant or provides reproducible evidence. Code contribution guidance will be established when implementation starts.

No project licence has been selected. The source and documentation must not be described as legally open or reusable until licence terms are published; contribution and security-reporting policies are also still pending.
