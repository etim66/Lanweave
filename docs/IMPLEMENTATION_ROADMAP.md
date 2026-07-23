# Implementation Roadmap

This roadmap is ordered by dependency gates, not calendar estimates. The project currently has no implementation. A gate is complete only when its behavior is documented, tested, and suitable as the dependency of the next gate.

## Gate 1: Wire And Framing

Define and prototype the fixed frame header, control and `DATA` frame kinds, strict JSON profile, limits, and incremental stream parser.

Exit criteria:

- Golden JSON vectors cover every fixed control message, including all four `pairing` records.
- `hello` boundary vectors retain exact JSON body bytes without frame headers: two 4,000-byte bodies produce the maximum valid 8,067-byte AAD, and either body at 4,001 bytes is rejected before PAKE work.
- Duplicate keys, invalid UTF-8, excessive depth, unsupported large integers, trailing data, unknown frame kinds, and invalid lengths fail deterministically.
- Every split and coalescing pattern yields the same frames without allocation above configured bounds.
- Version handling accepts exactly experimental version `1`; there is no capability path.

## Gate 2: Local Multi-File Pipeline

Implement the transfer and filesystem state machine without networking. Start with a one-file vertical slice, then generalize immediately to an immutable ordered 1..N manifest.

Exit criteria:

- Destination planning and index-0 preparation reject unsafe, duplicate, platform-equivalent, existing, and unpreparable names/resources before an accepting `transfer_response`; preparation failure produces rejection rather than post-acceptance failure.
- Files stream sequentially through bounded buffers and temporary files.
- Each attempted file produces at most one `file_result` when the connection remains usable; successful results follow verification and finalization, while failed results never publish the current partial.
- A later-file failure deletes its partial, retains the verified prefix, and prevents remaining files from starting.
- Multi-file success and later-file failure tests pass. One-file-only behavior does not clear this gate.

## Gate 3: TCP/TLS Transport

Carry the framing and transfer state machine over TLS 1.3, one transfer per connection. Generate a fresh ephemeral connection identity with the `rustls` 0.23, `tokio-rustls` 0.26, and `rcgen` 0.14 families. Before pairing, this is `Provisional TLS` and is suitable only for non-sensitive transport experiments; after pairing it is pairing-authenticated but remains experimental and not safe for sensitive use until Gate 4 clears.

Exit criteria:

- Partial reads/writes, split/coalesced frames, disconnects, cancellation, timeout, and backpressure are deterministic.
- Only TLS 1.3 is enabled; per-connection identities are not persisted or presented as authenticated device identities.
- The custom certificate verifier permits only the specified self-signed/name exception while still validating TLS 1.3 `CertificateVerify`; explicit rustls provider policy disables unwanted defaults, TLS 1.2, resumption, tickets, PSKs, early data, and key logging.
- One bounded writer path prevents frame interleaving and unbounded queue growth.
- No application ACK window, second metadata phase, second connection, transfer-complete message, or close handshake appears.

## Gate 4: Pairing Implementation And Audit

Implement the selected RFC 9382 SPAKE2-P256-SHA256-HKDF-HMAC profile behind a small adapter, with sender A, receiver B, the specified exporter invocation and SPAKE2 AAD composition, and mutual confirmation. The exact `hello` JSON body bytes, excluding frame headers, and exporter are AAD inputs for confirmation-key derivation, not `TT` inputs. Add a connection-independent ceremony manager for the receiver-generated code. Do not implement group arithmetic, add dummy PAKE work, or reopen profile selection.

Exit criteria:

- The receiver generates all 100,000,000 8-digit decimal values uniformly, preserves leading zeros, keeps ceremony secrets only in memory, and uses a 120-second monotonic deadline.
- One ceremony atomically allows five failed sender-confirmation verifications across reconnects and concurrent connections. In-flight pairing states and connection/source/CPU limits are bounded separately. Receiver verification of `cA` moves the ceremony to sender-confirmed; receiver pairing additionally requires `cB` to be flushed to TLS, and sender pairing requires verification of `cB` after `cA` is committed.
- Exact vectors cover leading-zero password-to-scalar mapping, identities `lanweave-v1-sender`/`lanweave-v1-receiver`, 32-byte big-endian padded `w`, eight-byte little-endian `TT` lengths and `Ke`/`Ka`/`KcA`/`KcB` splits, four-byte big-endian AAD length prefixes and order, exact `hello` bodies, and the exporter label `EXPORTER-Channel-Binding`, 32-byte output, and no-context invocation. Every identity, encoding, length, order, hello-byte, exporter-label/output/context, connection, and confirmation variation rejects.
- The sender-share, receiver-share, sender-confirmation, and receiver-confirmation `pairing` records pass RFC 9382 vectors, invalid-point tests, and independent interoperability tests.
- Specialist review accepts the exporter invocation and SPAKE2 AAD composition, including exporter equality, the RFC AAD bound, and split-MITM analysis.
- The PAKE implementation is independently audited, RFC-conformant, constant-time where required, and uses `zeroize` for secret material.
- Pairing completes before `transfer_request`; separate exact-manifest approval is required before `ready` and `DATA`.

No current crate clears this gate. A `pakery-spake2` 0.2.1 interoperability prototype also includes `pakery-core` and `pakery-crypto` with its `p256` and `spake2` features; audit and dependency review cover that full candidate-only surface, and the adapter discards `Ke`/`session_key` after confirmation. RustCrypto `spake2` is not acceptable. If the implementation and review requirements cannot be cleared, release remains blocked. Do not substitute custom cryptography.

## Gate 5: Discovery

Add minimal mDNS/DNS-SD advertisement and browsing, plus direct-address fallback. Discovery provides untrusted candidate endpoints and the `v=1` hint only.

Exit criteria:

- Stale, spoofed, duplicate, multi-interface, IPv4, and scoped IPv6 records are handled safely.
- Discovery loss does not alter or terminate an active connection or transfer.
- Direct address works when multicast is unavailable.

## Gate 6: Hardening And Release

Integrate CLI consent, limits, cross-platform filesystem behavior, fuzzing, fault injection, interoperability vectors, packaging, and documentation review.

Release gates:

- Multiple files transfer successfully in order; multi-file is mandatory.
- Fail-fast verified-prefix behavior passes on every supported platform.
- The pairing implementation and dependency audit, specialist exporter invocation and SPAKE2 AAD composition review, exact profile/RFC vectors, and adversarial tests pass in the shipped implementation.
- Parsers and filename mapping have no known crashing, escaping, or unbounded-allocation corpus case.
- `cargo audit` and `cargo deny` pass under the release policy.
- Licence, contribution, security-reporting, and experimental-security language are published before accepting contributions or distributing a release.

No sensitive release may be distributed until Gate 4 is cleared.

## Deferred Work

Directories, resume, parallelism, compression, overwrite or rename negotiation, trusted devices, algorithm agility, QUIC, mobile clients, and GUIs begin only after the MVP gates are complete.
