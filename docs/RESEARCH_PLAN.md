# Research Plan

Research is organized by release risk rather than a fixed week. Record the date, platform, versions, primary sources, reproducible steps, results, limitations, licence status, and affected decision for every experiment.

## Priority 1: PAKE Implementation And Composition

This is the primary security and release gate.

Questions:

- Does a candidate Rust implementation conform exactly to RFC 9382 SPAKE2-P256-SHA256-HKDF-HMAC, including point validation, exact identities, padded `w` encoding, eight-byte little-endian `TT` lengths, `Ke`/`Ka` and `KcA`/`KcB` key splits, HKDF, and HMAC confirmations?
- Does it pass RFC 9382 vectors and independent cross-implementation vectors for sender A and receiver B?
- Is it independently audited, maintained, constant-time where required, and capable of zeroizing code-derived and ephemeral secrets?
- Does `rustls` 0.23 expose equal 32-byte exporter material to both peers for label `EXPORTER-Channel-Binding` with no context, and does the exporter invocation and SPAKE2 AAD composition resist exporter substitution and split-MITM scenarios?
- Do four-byte big-endian `lp` values encode the AAD fields in exact label, exporter, sender-`hello`, receiver-`hello` order, using exact JSON body bytes without frame headers? Two 4,000-byte bodies produce the maximum valid 8,067-byte AAD; either body at 4,001 bytes must reject before PAKE work.
- Do the four records and confirmation ordering fail closed under replay, reflection, role confusion, invalid points, wrong codes, reconnects, and concurrent attempts?

The algorithm/profile is already selected; this research does not design a new PAKE or compare alternatives for protocol selection. Evaluate candidate crates as replaceable adapters. The prototype surface includes `pakery-spake2` 0.2.1 plus `pakery-core` and `pakery-crypto` with its `p256` and `spake2` features. It is very new, unaudited, and candidate-only, and the adapter must discard returned `Ke`/`session_key` after confirmation. RustCrypto `spake2` is not acceptable because it targets an old draft, is unaudited, and is probably not constant-time.

Outputs are reproducible RFC and profile vectors for leading-zero scalar mapping, exact identities, `w`, `TT` and key splits, AAD length prefixes/order, exact hello bodies, and the exporter invocation, plus negative vectors in which every component variation rejects. Also produce interoperability results, complete dependency provenance and audit evidence, an exporter invocation and SPAKE2 AAD composition brief, certificate-verifier findings, and specialist review. No prototype result can replace an independent implementation audit or cryptographic review.

## Priority 2: Ceremony Prototype

Prototype a manager that outlives connections and stores ceremony state only in memory. Test unbiased generation over the complete 8-digit decimal space, leading zeros, a 120-second monotonic deadline, atomic accounting of five failed sender-confirmation verifications across reconnects and concurrency, sender-confirmed state before `cB` flush, and at most one mutually paired connection and transfer. Bound in-flight PAKE states and connection/source/CPU work separately. Code sharing remains private and out of band. Do not add dummy PAKE work or claim timing/resource indistinguishability between protocol phases; test only the specified coarse remote authentication outcome after valid sender-confirmation verification.

## Priority 3: JSON And Frame Prototype

Build a disposable incremental parser for the proposed fixed header, JSON control payloads, and raw `DATA` payloads.

Test duplicate JSON keys, invalid UTF-8, depth and collection limits, large integers, trailing input, every frame kind, invalid/oversized lengths, and bounded allocation. Produce golden vectors suitable for another language. This work compares strict JSON library behavior; it does not benchmark CBOR or add codec negotiation.

## Priority 4: TCP/TLS Stream Prototype

Run two local processes using TLS 1.3 and fresh `rcgen` 0.14 identities. Deliberately split headers and payloads across writes, coalesce multiple frames, perform partial reads/writes, disconnect at every boundary, and apply bounded-channel backpressure. Confirm that one writer preserves DATA/`file_end` FIFO order and can discard queued DATA before a terminal control without an application ACK window or second connection. Verify `hello` and all four pairing records precede any manifest disclosure.

Before pairing, TLS in this prototype is `Provisional TLS`. After an integrated pairing it is pairing-authenticated, but remains experimental and must not be described as safe for sensitive use until Priority 1 and the independent audit are complete.

## Priority 5: Filesystem Safety Prototype

Test destination planning and temporary-file finalization on intended desktop platforms.

Cover duplicate and platform-equivalent names, case folding, Unicode normalization, reserved names, separators, symlinks, existing destinations, finalization races, disk-full behavior, source mutation, zero-byte files, and files larger than memory. Demonstrate that destination preparation precedes an accepting `transfer_response`, preparation failure rejects, and no `ready` is sent on that path. Also demonstrate multi-file success and a later-file failure that retains the verified prefix and removes the current partial.

## Priority 6: mDNS Prototype

Advertise `_lanweave._tcp.local.` with only the minimal `v=1` TXT hint. Test Linux, Windows, and macOS behavior for registration, firewall prompts, multiple interfaces, IPv4/IPv6, scoped addresses, stale records, conflicts, goodbye records, and multicast-disabled networks. Verify a direct-address fallback separately.

Discovery results influence UX and adapter selection, not authentication or active transfer state.

## Completion Criteria

- An independently audited RFC-conformant Rust PAKE implementation and dependency tree and specialist exporter invocation and SPAKE2 AAD composition review clear the release gate, or the gap remains explicitly release-blocking.
- RFC 9382, invalid-point, confirmation, exporter, split-MITM, reconnect, concurrency, and one-use ceremony tests are reproducible.
- JSON and frame behavior has reproducible golden and malformed-input vectors.
- TCP splitting/coalescing and backpressure behavior is demonstrated under faults.
- Filesystem behavior supports the required fail-fast multi-file semantics on target platforms.
- mDNS limitations and direct-address behavior are documented from real systems.
