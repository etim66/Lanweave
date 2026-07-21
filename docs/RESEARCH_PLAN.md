# First-Week Research Plan

## Evidence standard

The point of this week is to replace plausible assumptions with evidence. A crate name, README, or product page is not enough to settle a protocol decision.

For each result, record the version, date, platform, primary documentation, a reproducible experiment, known limitations, licence and maintenance signals, and the decision the result affects. Every library capability listed below still needs verification against current documentation and source.

## Day 1 — Discovery and comparable systems

| Topic | Questions | Evidence/tools to verify | Decision / output |
| --- | --- | --- | --- |
| mDNS on Linux/Windows/macOS | Registration APIs/daemons? interface and IPv6 behavior? conflict rename? TTL/goodbye? firewall prompt? | OS docs, packet captures, two-host matrix, reboot/interface-change tests | DD-001; discovery behavior report |
| Rust mDNS libraries | Pure Rust vs system service? async API? multi-interface? maintenance/licence? TXT limits? | Official crate/source docs and a tiny disposable spike (later implementation phase) | Library shortlist with gaps; no capability assumptions |
| Comparable systems | How do open systems handle discovery, consent, pairing, identity, chunks, resume, paths? | Public protocol/source/docs for LocalSend, KDE Connect, Syncthing, Warpinator, Magic Wormhole, croc, Snapdrop, PairDrop; public product docs for AirDrop, Nearby Share, Quick Share | Comparative matrix; ideas only, no proprietary copying |

Study names do not imply equivalence to Lanweave. Especially distinguish file-sharing products, synchronization systems, browser relays, and pairing tools.

## Day 2 — Transport and async runtime

| Topic | Questions | Evidence/tools to verify | Decision / output |
| --- | --- | --- | --- |
| Tokio networking patterns | Cancellation safety? bounded channels? split stream ownership? timeouts? graceful shutdown? | Tokio official docs/source/examples; faulted socket experiments later | Task/ownership design and cancellation checklist |
| TCP with rustls | Self-signed/raw-key support? TLS exporter? ALPN? peer certificate hooks? TLS 1.3 suites? key logging defaults? | rustls official docs/source and standards; minimal two-peer handshake design | DD-002/DD-004/DD-010 feasibility report |
| Quinn/QUIC | Current platform maturity? stream/backpressure API? migration? cert validation? firewall behavior? | Quinn official docs/source, RFCs, LAN benchmark plan | QUIC comparison update and revisit criteria |
| Secure framing | Incremental parser patterns, cancellation, size class, control priority | Standards/prior art, parser prototype plan, abuse cases | Frame header and parser invariants |

## Day 3 — Serialization and versioning

| Topic | Questions | Evidence/tools to verify | Decision / output |
| --- | --- | --- | --- |
| CBOR | Deterministic encoding support? duplicate key behavior? size/depth limits? cross-language libraries? | RFC 8949, official library docs/source, golden vectors | DD-003 codec profile draft |
| MessagePack/Protobuf/postcard/JSON | Evolution, canonical form, unknown fields, generated code, no-std/Rust coupling, integer fidelity | Primary specs and official tools; encode/decode sample schema across two languages | Scored alternative matrix |
| Version/capability model | How are unknown critical fields represented? transcript handling of unknowns? | Schema experiments and compatibility cases | Registry format and 1.0↔1.x fixtures |

## Day 4 — Cryptography and identity

| Topic | Questions | Evidence/tools to verify | Decision / output |
| --- | --- | --- | --- |
| Ed25519/X25519 Rust support | Audits/maintenance? key formats? zeroization? platform backends? validation pitfalls? | Official crate docs/source, security advisories, RFC test vectors | DD-004/DD-005 library shortlist |
| HKDF/AEAD | Misuse-resistant APIs? nonce sequencing? TLS exporter versus app AEAD? | RFCs, official library docs, known-answer tests | Key schedule requirements, not custom code |
| Token PAKE/proof | Which reviewed PAKE fits short receiver code, mutual transcript binding, Rust/other-language interop? | CFRG/RFC standards, cryptographer review, audited libraries | DD-010 decision memo; security gate |
| Secure local key storage | Windows Credential Manager/DPAPI, macOS Keychain, Linux Secret Service/headless fallback? | OS primary docs, permission/backup/rotation experiments | DD-013 platform matrix |

## Day 5 — Files, states, and testing

| Topic | Questions | Evidence/tools to verify | Decision / output |
| --- | --- | --- | --- |
| Streaming/hash large files | Async/sync file behavior? bounded buffers? source mutation? sparse files? SHA-256 performance? | Rust std/Tokio/crypto docs, benchmark plan on HDD/SSD, >RAM files | Chunk/window/hash benchmark plan |
| Cross-platform config dirs | Correct data/config/cache/key locations and permissions? | OS docs and a current cross-platform directory crate's official docs | Identity/config layout decision |
| State-machine patterns | Explicit enums/reducers vs library? compile-time states vs runtime persisted events? | Rust patterns/crate docs, property-test ergonomics | Central reducer API decision |
| Protocol fuzzing | Parser harnesses? corpus/minimization? sanitizer support on CI platforms? | cargo-fuzz/libFuzzer and alternatives' official docs (verify) | Fuzz target inventory and CI plan |

## End-of-week outputs

- Evidence-linked decision updates for DD-001 through DD-013.
- mDNS/firewall three-platform result matrix.
- Serialization golden-schema comparison and framing threat review.
- A security-review brief for TLS/PAKE/transcript binding.
- Library shortlist with versions, licences, maintenance, audit/advisory notes—no fabricated claims.
- A reduced Week 2 scope if any security gate remains unresolved.

## Priority order

1. Pairing/TLS/identity construction and token-guessing resistance.
2. Canonical serialization/transcript representation.
3. Cross-platform mDNS/firewall feasibility.
4. Key storage and filesystem-safe APIs.
5. TCP framing/backpressure and large-file hashing.

The first three can invalidate message fields or the transport profile and therefore precede implementation scaffolding.
