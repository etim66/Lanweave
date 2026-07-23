# Testing Strategy

The test suite must describe version 1 wire and transfer behavior independently of the initial module layout. Security tests demonstrate rejection of known bad cases; they do not establish that an unaudited implementation or unreviewed composition is secure.

## JSON Control Tests

- Golden encode/decode vectors for `hello`, `transfer_request`, `transfer_response`, `ready`, `file_end`, `file_result`, `cancel`, and `error`.
- Golden encode/decode vectors for all four fixed `pairing` records: sender A share, receiver B share, sender A confirmation, and receiver B confirmation.
- Preserve exact sender and receiver `hello` JSON body bytes, excluding frame headers, across insignificant whitespace and field-order variants. Test two 4,000-byte bodies and the resulting 8,067-byte AAD as valid; either individual body at 4,001 bytes is `invalid_message` before PAKE work.
- Duplicate keys at every object level are rejected, including duplicates that a generic map would overwrite.
- Invalid UTF-8, excessive nesting, excessive arrays/maps/strings, trailing JSON values, unsupported escapes, and malformed numbers are rejected within fixed work bounds.
- Integers outside the protocol's explicitly supported range are rejected before conversion or arithmetic; large integer literals must not silently round through floating point.
- Unknown required fields, missing fields, wrong types, invalid enum strings, and protocol versions other than exactly `1` are rejected deterministically.

## Frame Tests

- Golden vectors for control and `DATA` frame kinds and boundary payload lengths.
- Every possible split point across headers and payloads produces the same result as contiguous input.
- Multiple coalesced frames parse in order, including alternating JSON control and raw `DATA` frames.
- Outbound DATA and `file_end` remain FIFO; terminal controls discard queued nonterminal frames before bypassing them.
- Unknown kinds, malformed headers, zero/maximum/maximum-plus-one lengths, premature EOF, and lengths inconsistent with message policy fail safely.
- Declared lengths are checked before allocation; slow or fragmented input cannot grow memory without bound.
- Raw `DATA` bytes are never passed to JSON decoding, and control payload bytes are never written as file data.

## State And Multi-File Tests

- No `transfer_request` or other manifest disclosure is accepted before mutual pairing confirmation.
- Pairing success does not imply file consent. Exact whole-manifest review and approval is separately required before `ready` or `DATA`.
- Destination preparation occurs after local whole-manifest approval but before an accepting `transfer_response`; every preparation failure sends a rejection, never acceptance or `ready`.
- The approved manifest is immutable; reordered, replaced, added, or removed entries are rejected.
- Successful 1-file, 2-file, and many-file manifests process in exact order.
- Each uninterrupted successful file has one `file_end` and one verified `file_result`; a reportable current-file failure has one failed result. Interruption may prevent a result, and no file has more than one.
- Failure in the second or later file deletes that current partial, retains all and only the verified prefix, emits failure once, and prevents later files from opening.
- Zero-byte, multi-chunk, larger-than-memory, incorrect length, hash mismatch, source mutation, disk-full, disconnect, timeout, and cancellation cases have deterministic outcomes.
- DATA before pairing-authenticated `ready`, DATA arriving in a state reserved for another manifest index, excess DATA, and DATA after failure are rejected without unsafe final files.
- The final verified `file_result` completes the protocol; tests ensure no transfer-complete or close-handshake dependency appears.

## Filesystem Tests

- Reject duplicate names and destination names equivalent under each supported platform's rules.
- Reject existing files, directories, symlinks, and other destination entries; never overwrite or negotiate a rename.
- Cover separators, absolute paths, traversal, reserved names, control characters, Unicode normalization, case folding, trailing dots/spaces, and maximum lengths.
- Exercise symlink and destination-creation races, no-follow behavior, temporary-file permissions, atomic finalization assumptions, cleanup after process failure, and checked disk/size arithmetic.

## Property And Fuzz Tests

- Fuzz the frame decoder, strict JSON decoder, semantic validators, transfer state reducer, and destination-name mapper with retained minimized corpora.
- For arbitrary TCP segmentation and coalescing, decoded frame sequences are invariant or produce one bounded error.
- For arbitrary state events, no file bytes are accepted before exact approval and pairing-authenticated readiness.
- A finalized file always has the declared length and digest; a partial file is never presented as final.
- Counts, lengths, totals, manifest indices, and progress never wrap or exceed configured bounds.
- No accepted manifest name escapes the receiver-selected destination or aliases another accepted name.

## Integration And Fault Tests

- Run separate sender and receiver processes over loopback and real LANs with partial writes, disconnects at every protocol boundary, cancellation races, and bounded queues.
- Test IPv4, IPv6, multiple interfaces, stale/spoofed mDNS, multicast loss, direct-address fallback, and discovery disappearance during an active transfer.
- Test supported platform pairs and filesystem combinations, not only same-platform loopback.
- Measure throughput, CPU, memory, queue bounds, many-small-file behavior, and files larger than memory without weakening correctness limits.

## Ceremony Tests

- Use statistical and property tests to detect modulo bias across the full 8-digit decimal code space; include deterministic fixtures for `00000000`, leading-zero values, and `99999999`.
- Test immediately before, exactly at, and immediately after the 120-second monotonic deadline without consulting wall-clock time.
- Verify one through five failed sender-confirmation verifications across reconnects, reject every later verification, and ensure malformed input, disconnects, connection caps, and CPU/rate-limit rejection neither count nor reset this budget.
- Independently test bounds for active connections, in-flight PAKE states, per-source admission, and CPU work so attempts that do not reach sender-confirmation verification cannot consume unbounded resources.
- Race failed sender-confirmation verification, sender-confirmed transition, expiry, cancellation, and reconnects from many tasks. Atomic state permits no more than five counted failures and at most one sender-confirmed ceremony; mutual pairing still requires receiver-confirmation commit/flush and sender verification.
- Verify sender-confirmed, expiry, exhaustion, or cancellation makes the memory-only ceremony unavailable, cancels attached PAKE states, and best-effort zeroizes the code and derived secrets without persistence. Do not insert dummy PAKE work for unavailable states or treat coarse wire outcomes as proof of timing/CPU indistinguishability.

## Security Profile Tests

- Run RFC 9382 SPAKE2-P256-SHA256-HKDF-HMAC vectors and independent interoperability vectors with sender A and receiver B.
- Add exact application-profile vectors for SHA-512 password mapping with `00000000` and other leading-zero codes; identities `lanweave-v1-sender` and `lanweave-v1-receiver`; 32-byte big-endian padded `w`; eight-byte little-endian `TT` lengths/order; SHA-256 `Ke`/`Ka` 16-byte splits; and HKDF-SHA256 `KcA`/`KcB` 16-byte splits. `Ke` and candidate-library `session_key` are discarded after confirmation.
- Add exact AAD vectors for four-byte big-endian `lp` encoding in label, exporter, sender-`hello`, receiver-`hello` order. The hello inputs are exact JSON body bytes without frame headers and are used only in AAD for confirmation-key derivation, never inserted into `TT`.
- Verify the RFC 9266 exporter invocation uses exact label `EXPORTER-Channel-Binding`, requests exactly 32 bytes, and supplies no context. Positive peers match; changing the password bytes, either identity, `w` padding, any `TT` length/order, AAD prefix/order/body byte, exporter label/output length, adding or varying a context in negative tests, or using another TLS connection makes confirmation reject.
- Cover valid records plus identity, low-order, off-curve, malformed, and non-canonical point inputs as applicable; every invalid point fails before confirmation or key use.
- Test all four pairing records independently for malformed input, wrong order, replay, reflection, role confusion, `TT` mutation, AAD mutation, and missing/incorrect HMAC confirmation.
- Test confirmation commit boundaries under partial writes, cancellation, and disconnect: sender pairing requires `cA` committed and `cB` verified; receiver pairing requires valid `cA` and `cB` committed/flushed to TLS. Verification before the flush is only sender-confirmed ceremony state, not mutual pairing.
- Use split-MITM and exporter-substitution harnesses; mutual confirmation must fail when the PAKE peers are attached to different TLS connections.
- Verify the certificate verifier's exact exception: it may accept the expected fresh self-signed chain/name, but validates certificate structure and policy, supported signature schemes, and the TLS 1.3 `CertificateVerify` signature and rejects invalid possession. No verifier path may return unconditional signature success.
- Pin and test the rustls crypto-provider and feature policy rather than inheriting defaults. Verify unused default provider/features are disabled and cannot re-enable TLS 1.2, resumption, session tickets, PSKs, early/half-RTT data, unapproved algorithms, or release key logging.
- Verify only TLS 1.3 is enabled, ephemeral identities differ across connections, and every transport private-key representation is erased where the selected APIs permit; inability to demonstrate erasure remains a review gate.
- Scan structured logs, errors, traces, panic output, and test artifacts for the decimal code, PAKE scalars, derived keys, confirmation keys, and exporter material.
- Test `zeroize` paths on success, every error, timeout, cancellation, and panic boundary that can be safely exercised.
- Run `cargo audit` and `cargo deny` in CI and before release; record any exception explicitly rather than silently allowing it.

`Provisional TLS` applies only before pairing. After pairing, tests and diagnostics call the connection pairing-authenticated, while documentation and release checks continue to mark it experimental and not safe for sensitive transfer until the implementation/dependency audit and exporter invocation and SPAKE2 AAD composition review pass.

## Release Gates

- All JSON/frame golden vectors and malformed-input suites pass.
- Multi-file success and later-file fail-fast behavior pass on every supported platform.
- Fuzzing has no known crashing, escaping, or unbounded-allocation corpus case.
- The fixed security profile's exact RFC/application vectors and adversarial variation tests pass in an independently audited implementation and dependency tree, and specialist review accepts the exporter invocation and SPAKE2 AAD composition.
- Ceremony bias, expiry-boundary, attempt, reconnect, concurrency, one-use, secret-log, and zeroization tests pass.
- `cargo audit` and `cargo deny` pass under the documented policy.
- No release depends on CBOR, capability negotiation, application ACK windows, or single-file-only behavior.
