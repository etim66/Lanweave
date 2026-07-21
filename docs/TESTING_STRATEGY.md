# Testing Strategy

## Principles

I want the test suite to describe the protocol, not mirror whichever module layout the first implementation happens to use. Every wire type needs a normal case and cases for boundaries, malformed data, unknown fields, the wrong state, and the wrong protection level.

Deterministic clocks and random-number fixtures are useful in tests, but they must not leak into production builds. Security tests show that known bad inputs are rejected; they are not evidence that the whole system is secure.

## Test layers

| Layer | Scope and important scenarios |
| --- | --- |
| Unit | ID/limit/name/version/capability validation; timeout arithmetic; progress; error mapping; transcript builder |
| Property-based | Arbitrary event sequences never bypass consent/session; frame split/coalescing equivalence; checked length arithmetic; name mapping containment |
| Serialization | Golden CBOR vectors in at least two independent implementations; canonical round trips; unknown optional/critical fields; enum boundaries |
| Frame parser | Every split point, many frames/read, partial EOF, bad magic/version/flags, 0/max/max+1 length, slow headers, allocation bounds |
| Malformed message | Duplicate fields, excessive nesting/counts, invalid UTF-8, trailing bytes, overflow, ID mismatch, unknown critical type |
| State machine | Every valid transition, every message in every invalid state, roles, timers, duplicate/idempotent cancel/close, terminal-state immutability |
| Integration | In-memory and loopback peers through negotiation/reject/pair/transfer/fail; real TLS/profile when settled |
| Multi-process | Separate CLI processes, concurrent requests, restart/stale instance, cancellation/signals, port conflicts |
| LAN/device | Linux/Windows/macOS pairs, IPv4/IPv6/link-local, multiple interfaces, firewall, AP isolation, sleep/wake, mDNS loss |
| Fault injection | Packet loss/delay/reorder at network layer, partial writes, disconnect every protocol step/chunk offset, disk-full, permissions, source mutation |
| File cases | Zero byte, boundary chunks, >RAM file, very large declared length, collisions, illegal names, Unicode/control names, symlink races |
| Fuzz | Frame decoder, CBOR decoder/schema, message validator, event reducer, filename mapper, transcript canonicalizer |
| Security | Replay, MITM harness, downgrade, token attempt/expiry, nonce/sequence duplicates, forged ACK/hash, log-secret scanning, flood limits |
| Performance | Discovery scale, handshake rate, throughput/CPU/RSS, buffer/window/chunk matrix, SHA-256 cost, many small files |
| Compatibility | 1.0↔1.0; future 1.0↔1.1/1.5; 1.x↔2.0 rejection; codec/frame golden corpus retained permanently |

## Property invariants

- No generated event sequence reaches `Transferring` without prior receiver acceptance, token success, key exchange, and two-sided session confirmation for the same IDs/transcript.
- Every accepted `FileChunk` is protected, belongs to the active file/transfer/session, starts at the expected offset, and stays within declared length.
- Any finalized file has received exactly its declared bytes and matches declared SHA-256.
- Token success occurs at most once; expiry/cancel/exhaustion makes all later submissions fail.
- Terminal states never transition back to active states.
- For arbitrary byte-stream segmentation, parsing yields the same frames as concatenated input or one bounded error; memory stays under configured bound.
- No accepted remote filename maps outside the receiver-selected directory.
- Negotiated version/capabilities are a subset of both offers; changing an offered/selected security value changes transcript verification.
- Per-direction AEAD nonce/sequence is never reused within a session, and duplicate/out-of-order protected records fail per profile.
- Counts, offsets, totals, and progress use checked arithmetic and never wrap.

## Scenarios to write first

1. Successful zero-byte and multi-chunk single file.
2. Receiver rejection creates no token/temp file.
3. Five bad token attempts consume challenge; sixth cannot succeed.
4. Token expires at exact monotonic boundary.
5. Version 1.0 and 2.0 exchange no transfer message.
6. MITM changes negotiation/identity/ephemeral/session ID and confirmation fails.
7. Data before `TransferReady` or secure confirmation is rejected and no file appears.
8. Disconnect at each byte split and state cleans up bounded resources.
9. Claimed oversized frame is rejected before payload allocation.
10. Disk becomes full after readiness; partial is not finalized.
11. Source changes during transfer; operation fails rather than mixing silently.
12. Hash mismatch preserves existing destination and removes/quarantines temp per policy.
13. Cancel races with chunk/write/complete and has one deterministic terminal result.
14. mDNS spoof redirects endpoint but cannot authenticate secure session.

## Suggested Rust ecosystem tools (verify before selection)

The likely toolbox includes `cargo test`, a property-testing library such as `proptest` or `quickcheck`, `cargo-fuzz` with libFuzzer, Tokio's time controls, `criterion`, and network-fault tools such as Linux `tc netem` in isolated CI. Sanitizers, Miri, and Loom may help with the parts they support.

These are candidates, not dependencies chosen by this document. I still need to check their current capabilities, licences, maintenance, and platform support in Week 1.

## CI and release gates

- Fast unit/property/golden tests on every change.
- Cross-platform integration matrix on protected branches.
- Scheduled fuzz and long/large-file/fault suites with retained corpora.
- No release with failing compatibility vectors, secret-in-log tests, unsafe unresolved crypto profile, or unreviewed parser changes.
- Performance regressions use ranges and environment baselines, not flaky absolute LAN timing.

Manual two-device testing complements but never replaces deterministic protocol tests.
