# Design Decisions

This is my working decision log for the project. An **Accepted** decision is part of the current baseline. **Proposed** means it is the direction I intend to take, but I still want evidence from implementation work. **Needs Research** marks a question that blocks a final choice. **Deferred** keeps something outside the first release, and **Rejected** records an option I considered but do not plan to use in this context.

The summary table is meant for scanning. The notes below explain why each choice is here and what would make me change it.

## Decision log

| ID | Decision | Status | Chosen direction |
| --- | --- | --- | --- |
| DD-001 | Discovery | Proposed | mDNS/DNS-SD using `_lanweave._tcp.local.` |
| DD-002 | Initial transport | Proposed | TCP with TLS 1.3; leave QUIC for later |
| DD-003 | Serialization | Proposed | CBOR with deterministic/canonical profile |
| DD-004 | Device identity | Proposed | Persistent Ed25519 |
| DD-005 | Ephemeral agreement | Needs Research | X25519 within reviewed TLS/PAKE/handshake profile |
| DD-006 | AEAD | Proposed | ChaCha20-Poly1305 default; AES-GCM negotiable |
| DD-007 | File hash | Accepted | SHA-256 over exact file bytes |
| DD-008 | Identifiers | Proposed | UUIDv4 random IDs; consider UUIDv7 only for local records |
| DD-009 | Token origin | Accepted | Receiver-generated, one-time, short-lived |
| DD-010 | Token verification | Needs Research | Prefer reviewed PAKE/proof; plaintext submission experimental only |
| DD-011 | Connection topology | Proposed | One protected framed TCP connection for control/data |
| DD-012 | Acknowledgements | Proposed | Cumulative application checkpoints, TCP reliability underneath |
| DD-013 | Identity storage | Needs Research | OS key store where available; restricted-file fallback |
| DD-014 | Workspace | Proposed | Start protocol/core/CLI; split internal modules later |
| DD-015 | Product interface | Accepted | CLI-first reference implementation |
| DD-016 | Service type | Accepted | `_lanweave._tcp.local.` |

## Records

### DD-001 — mDNS/DNS-SD

**Status: Proposed.**

Lanweave needs a way to find peers without asking users to type addresses. I looked at mDNS/DNS-SD, SSDP, UDP broadcast, direct IP entry, Wi-Fi Direct, Bluetooth, and a central rendezvous service.

mDNS/DNS-SD is the best fit for the first release because it already models local services, works with IPv4 and IPv6, and does not introduce a server. It is not universally reliable: guest networks may isolate clients, firewalls may block it, and advertisements can leak metadata or be forged. I will keep direct address entry in mind as a fallback and revisit this choice if the three desktop platforms behave too differently in practice.

### DD-002 — TCP/TLS versus QUIC

**Status: Proposed.**

The first CLI needs a reliable encrypted connection, not every transport feature I may want later. The realistic choices are TCP with TLS 1.3 and QUIC, which also uses TLS 1.3.

I plan to start with TCP and TLS. It is easier to deploy, inspect, and debug, and one ordered stream is enough for the single-file milestone. I accept that this means application framing, head-of-line blocking, and a reconnect when the active interface changes. The open risk is how pairing authenticates the TLS channel; that question must be settled before this decision becomes final. I will revisit QUIC when parallel file streams or mobile connection migration become real requirements rather than anticipated ones.

### DD-003 — CBOR serialization

**Status: Proposed.**

Messages need to be compact, bounded, evolvable, and practical outside Rust. I considered CBOR, MessagePack, Protocol Buffers, postcard, and JSON. JSON is pleasant to inspect but awkward for binary chunks and canonical signing. Postcard is attractive in Rust but weaker as a cross-language protocol choice. Protobuf brings a strong schema toolchain, while MessagePack and CBOR keep the data model more visible.

My current preference is a deliberately small CBOR profile, with JSON-like examples in the docs. That choice comes with work: I must define numeric keys, canonical encoding, duplicate-field handling, depth and length limits, and independent test vectors. I cannot assume that a Serde round trip is the protocol. I will reconsider Protobuf if canonical CBOR proves difficult to implement consistently in more than one language.

### DD-004 — Ed25519 identity

**Status: Proposed.**

Each installation needs a stable key so a session can be tied to the same device identity over time. The alternatives I considered are Ed25519, ECDSA P-256, relying only on TLS certificates, or avoiding persistent identity altogether.

Ed25519 is my starting point because its keys and signatures are compact and good library support is available. I still need to specify storage, rotation, fingerprints, and how the key appears in the TLS or handshake profile. A stolen key can impersonate the device, and possession of the key is not proof that a user approved a transfer. I will revisit the algorithm if platform key stores or the final TLS profile strongly favor a different reviewed scheme.

### DD-005 — X25519 ephemeral agreement

**Status: Needs Research.**

I want forward secrecy, but choosing an elliptic-curve operation is only a small part of choosing a secure handshake. The options are X25519, P-256 ECDHE, letting TLS 1.3 own the exchange, or using a PAKE that includes key agreement.

X25519 is the preferred primitive if the selected protocol needs one. I will not build a handshake by stitching X25519, signatures, and the pairing token together myself, and I will never use its raw shared secret as a traffic key. This decision stays open until I have a reviewed construction, a clear HKDF and transcript design, explicit key confirmation, and interoperability test vectors.

### DD-006 — ChaCha20-Poly1305

**Status: Proposed.**

For application-level authenticated encryption, the two practical candidates are ChaCha20-Poly1305 and AES-GCM. If TLS owns record encryption, I should use TLS's cipher-suite negotiation instead of adding another layer by habit.

ChaCha20-Poly1305 is the proposed default because its performance is dependable on machines without AES acceleration. AES-GCM can remain a negotiated option. Either choice requires strict key separation and nonce accounting; nonce reuse is a serious failure in both algorithms. Benchmarks, platform requirements, or compliance needs may justify changing the default.

### DD-007 — SHA-256 file hashing

**Status: Accepted.**

The receiver needs a digest of the exact bytes it wrote. I considered SHA-256, BLAKE3, a per-chunk Merkle tree, and relying on transport AEAD alone.

Version 1 uses SHA-256 because it is widely available and easy for independent implementations to agree on. It can be computed while streaming, although a high-assurance second pass costs another full read. The digest does not authenticate the sender on its own, and it does not solve the problem of a source file changing mid-transfer. A future resume or content-addressing design may add chunk hashes or a tree, but the whole-file SHA-256 should remain available for compatibility.

### DD-008 — UUIDv4 versus UUIDv7

**Status: Proposed.**

Requests, sessions, transfers, files, and messages all need identifiers that can be created independently. UUIDv4 and UUIDv7 are the two candidates.

I prefer UUIDv4 on the wire. It does not reveal a creation time and does not depend on a correct clock. The cost is that IDs do not sort chronologically, which is acceptable for the protocol. UUIDv7 may still be useful in a local database where ordering matters and timestamp leakage is understood. UUIDs are never treated as secrets or proof of authenticity.

### DD-009 — receiver-generated token

**Status: Accepted.**

The pairing ceremony should follow the direction of authorization: the receiving device is the one deciding whether it will accept writes. I considered a sender-generated token, a receiver-generated token, and a short code shown on both devices for comparison.

The receiver will generate the token after the user approves the request. The sender then types the value shown on the receiver. This is easy to explain and ties the token to the receiver's decision, but it does not by itself prevent relay or social-engineering attacks. I may revisit a two-sided comparison flow if usability or accessibility testing shows it is clearer.

### DD-010 — token submission/proof

**Status: Needs Research.**

A human-entered token has low entropy. Sending it directly is simple, but it exposes the token to whichever endpoint terminates the provisional channel and provides poor protection against a badly designed MITM flow. I considered direct submission over a confidential channel, a simple challenge hash, and a proper PAKE or proof protocol.

The draft keeps direct submission only so an experimental prototype has an explicit schema. It is not the target design for a security-stable release. A salted hash is also not enough because an observer can test guesses offline. I want a reviewed PAKE or proof that binds the token to the channel and transcript. The fields may change before the 1.0 wire format is frozen; this decision cannot be accepted without cryptographic review and shared test vectors.

### DD-011 — one connection

**Status: Proposed.**

Control messages and file data can share one framed stream, use separate TCP connections, or use separate QUIC streams.

For the first implementation, one TCP/TLS connection is easier to authenticate, correlate, and close cleanly. The writer must prioritize control frames and bound queued file data so that cancellation is not stuck behind megabytes of chunks. Head-of-line blocking is the main cost. I will revisit the topology if measurements show it is a real problem or if a QUIC transport profile is added.

### DD-012 — checkpoint acknowledgements

**Status: Proposed.**

TCP only tells the sender that bytes reached the other network stack; it does not say that Lanweave accepted or wrote them. I considered acknowledging every chunk, acknowledging ranges, using cumulative checkpoints, or relying on TCP alone.

The initial design uses a cumulative highest-contiguous offset after 4 MiB, after one second, and at the end of a file. A negotiated receive window keeps in-flight application data bounded. These acknowledgements add traffic, but they give useful progress and backpressure without a stop-and-wait loop. They must not be described as durable writes unless the receiver has actually performed the required flush. Resume or out-of-order QUIC streams would require a richer scheme.

### DD-013 — identity storage

**Status: Needs Research.**

The device identity key must survive restarts without being left casually readable. The choices are an operating-system credential store, a permission-restricted file, or an external agent or hardware-backed key.

I would prefer native key storage where it works and a clearly documented file fallback where it does not, especially on headless Linux. That means platform code, migrations, backup behavior, and failure handling all need attention. No software storage option protects against malware running as the user. I will settle this only after testing Windows, macOS, desktop Linux, and a realistic headless Linux setup.

### DD-014 — workspace structure

**Status: Proposed.**

The architecture identifies seven useful responsibilities, but that does not mean I need seven crates on day one. I considered the full split, a three-crate workspace, and a single crate.

I will start with `lanweave-protocol`, `lanweave-core`, and `lanweave-cli`. The core crate will keep discovery, transport, cryptography, and transfer in separate modules so they can move later without being tangled. The risk is that `lanweave-core` becomes a grab bag; dependency rules and module ownership should prevent that. I will split a module when reuse, platform isolation, fuzzing, or compile-time boundaries give me a concrete reason.

### DD-015 — CLI first

**Status: Accepted.**

The first reference implementation could be a CLI, a GUI, or a library with no user interface. A GUI would make discovery and approval friendlier, but it would also add platform and presentation work before the protocol has settled.

I am starting with a CLI because it makes the states and errors visible and is straightforward to exercise in integration tests. This choice does not make terminal UX an afterthought: peer names need safe rendering, secrets must not land in shell history, and prompts must be unambiguous. GUI work can begin after the secure single-file path is stable.

### DD-016 — service type

**Status: Accepted.**

DNS-SD needs one service type that every implementation can browse for. The project-specific TCP label is `_lanweave._tcp.local.`.

I will use that value consistently for the initial TCP profile. It also makes the use of TCP visible in discovery. A future QUIC profile cannot quietly reuse it; it will need its own service-type decision. Before the first public release, I will verify registration and namespace requirements and change the label if the relevant standards require it.
