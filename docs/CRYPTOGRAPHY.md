# Cryptography

## Release-blocking status

Version 1 selects the experimental profile in this document so that its wire behavior can be implemented and reviewed. Selection is not approval. The application-specific password mapping, the entire TLS/SPAKE2 composition, and any implementation remain release-blocking until specialist review, RFC vectors, composition tests, and adversarial tests pass. Lanweave MUST NOT claim that safe production code currently exists.

The Lanweave profile is fixed and not negotiated: RFC 9382 SPAKE2-P256-SHA256-HKDF-HMAC over TLS 1.3. There is no Lanweave, application, or PAKE algorithm negotiation. Standard TLS 1.3 cipher-suite and key-exchange negotiation MAY occur only within one pinned, reviewed `rustls` cryptographic-provider policy; peers configure only TLS 1.3 and MUST NOT use another provider, protocol-version fallback, or application-level fallback. The sender is SPAKE2 Party A and the receiver is Party B. Their exact identity strings are ASCII `lanweave-v1-sender` and ASCII `lanweave-v1-receiver`, respectively.

## Pairing code

For each pairing ceremony, the receiver generates an 8-character decimal string uniformly with the operating system CSPRNG. Generation MUST use rejection sampling rather than modulo reduction, and leading zeroes MUST be preserved. The code:

- is kept in memory only and MUST NOT be logged or persisted;
- expires 120 seconds after generation according to a monotonic clock;
- is one-use and allows at most five failed admitted sender-confirmation verifications total across all connections;
- is shared privately out of band and MUST NOT appear in any wire message.

Expiry, consumption, verification admission, and failed-attempt accounting MUST be atomic across concurrent connections. Implementations MUST prevent concurrent exchanges from exceeding the five-verification budget, including verifications already admitted. Connections, malformed input, share processing, and other PAKE work do not count as failed attempts. A successful sender confirmation atomically consumes the code. Expiry or exhaustion also makes it unusable. Implementations MUST impose finite global, per-source, and per-ceremony bounds on in-flight pairing states and rate-limit both connections and expensive pairing work.

The SPAKE2 password scalar is defined by this application mapping:

```text
digest = SHA-512(ASCII("Lanweave SPAKE2 code v1") || exact_8_ASCII_digits)
w = big_endian_integer(digest) mod P-256_group_order
```

No separator, terminator, normalization, integer parsing, or removal of leading zeroes is performed. This mapping is application-specific and requires specialist cryptographic review before release.

## TLS profile

The peers complete TLS before sending application data. The receiver is the TLS server and creates a fresh, in-memory ECDSA P-256 key and certificate for every connection. The server sends exactly one well-formed DER-encoded, self-signed X.509 certificate and no intermediates. The certificate self-signature MUST verify under its own subject public key as a shape check, not as an identity or trust check. Its subject public-key information MUST be ECDSA P-256, and the only permitted TLS `CertificateVerify` signature scheme is `ECDSA_NISTP256_SHA256`. TLS uses the `rustls` TLS 1.3 implementation and the fixed ALPN value `lanweave/1`.

The sender's provisional verifier enforces that certificate shape and key profile but intentionally does not use trust-chain, DNS name, SAN, validity-period, or EKU checks as peer-identity checks; SPAKE2 supplies peer authentication. This exception MUST NOT bypass TLS 1.3 `CertificateVerify`. A `rustls` verifier MUST fail every TLS 1.2 signature-verification entry point, advertise only `ECDSA_NISTP256_SHA256`, and verify the TLS 1.3 handshake signature against the certificate SPKI through the pinned provider's `rustls` TLS 1.3 signature-verification helper. Returning success without that verification is non-conforming.

TLS 1.2, 0-RTT, resumption, session tickets, and early or half-RTT application data are forbidden. The ordinary TLS 1.3 handshake, including TLS Finished verification, MUST complete. Each peer MUST then check that the negotiated protocol is exactly TLS 1.3 and ALPN is exactly `lanweave/1`; otherwise it closes before any Lanweave frame. These checks and standard TLS 1.3 cipher-suite or key-exchange selection are not Lanweave negotiation.

After the handshake, each peer derives exactly 32 bytes with the RFC 9266 TLS exporter using label `EXPORTER-Channel-Binding` and no context. Pairing aborts if exporter derivation fails or the peers do not use the exporter from this exact TLS connection.

## SPAKE2 binding

The SPAKE2 additional authenticated data (AAD) is the following byte string, where `lp(x)` is a four-byte unsigned big-endian byte length followed by exactly `x`:

```text
lp(ASCII("lanweave-pairing-v1")) ||
lp(tls_exporter_32_bytes) ||
lp(exact_sender_hello_JSON_body_bytes) ||
lp(exact_receiver_hello_JSON_body_bytes)
```

The exact JSON frame body bytes are retained as sent and received, including insignificant whitespace and object field order. They MUST NOT be parsed and reserialized for this binding. Each exact `hello` JSON body, including syntax and whitespace, MUST be at most 4,000 bytes. The four `lp` prefixes, 19-byte label, and 32-byte exporter contribute exactly 67 bytes, so AAD is at most `67 + 4,000 + 4,000 = 8,067` bytes, within the RFC 9382 maximum of 8,176 bytes. A peer MUST enforce the 4,000-byte limits and MUST NOT pass longer AAD to the PAKE.

The adapter supplies this byte string as RFC 9382 AAD for confirmation-key derivation in the same order on both peers; it is not inserted into the RFC transcript `TT`. The application `lp` encoding is four-byte unsigned big-endian and is used only in this AAD construction.

The exporter and AAD bind pairing to one direct, completed TLS connection. A split man-in-the-middle terminates two TLS connections and therefore obtains different exporters; sender confirmation cannot validate at the receiver. The soundness of this exporter/SPAKE2 composition remains a release-blocking specialist-review item.

## RFC 9382 transcript and keys

The SPAKE2 transcript `TT` and ciphersuite encodings are otherwise exactly RFC 9382. In particular:

- the Party A and Party B identities in `TT` are the exact ASCII byte strings `lanweave-v1-sender` and `lanweave-v1-receiver`;
- every RFC `TT` component has the RFC eight-byte unsigned little-endian length prefix; the application's four-byte `lp` encoding never appears in `TT`;
- `w` is encoded in `TT` as a big-endian integer padded to exactly 32 bytes, preceded by its RFC eight-byte little-endian length;
- `SHA-256(TT)` is split without further transformation into the first 16 bytes `Ke` and the final 16 bytes `Ka`;
- confirmation keys are `HKDF-SHA256` with input keying material `Ka`, nil salt, `info = ASCII("ConfirmationKeys") || AAD`, and output length 32 bytes, split into the first 16 bytes `KcA` and final 16 bytes `KcB`;
- Party A and Party B confirmations use their respective RFC confirmation keys and the full 32-byte HMAC-SHA256 outputs; tags are not truncated; and
- `Ke` is unused by Lanweave and is discarded with best-effort zeroization. It is never an application encryption key.

## Exchange and confirmation

Pairing uses exactly four JSON records in this order:

1. sender Party A share;
2. receiver Party B share;
3. sender Party A confirmation;
4. receiver Party B confirmation.

Shares are the RFC 9382 P-256 elements encoded as 65-byte SEC1 uncompressed points. Confirmations are the suite's 32-byte HMAC-SHA256 key-confirmation tags. Both use canonical unpadded base64url in the `data` field. Message state and direction determine the role; no role, code, algorithm, or profile selector is sent.

RFC 9382 key confirmation is mandatory in both directions. The receiver MUST validate the sender confirmation before sending its own and MUST compare confirmation tags without data-dependent timing. It atomically consumes the code when sender confirmation succeeds. A confirmation is committed only when its entire framed confirmation has been accepted by the TLS writer and the corresponding flush has completed successfully. Commitment is not proof that the peer received the frame. The sender succeeds only after its confirmation has been committed and the receiver confirmation verifies. The receiver succeeds only after the sender confirmation verifies and its own confirmation has been committed. If the final confirmation is lost, the consumed code makes the sender fail safely rather than treating an unconfirmed pairing as successful.

TLS remains the sole record layer before and after pairing. SPAKE2 output is used only for RFC 9382 key confirmation. It MUST NOT be used for file encryption, an application AEAD, or additional traffic keys.

Secret cleanup is best-effort zeroization, not a guarantee of physical erasure: compiler, allocator, library, and operating-system copies may remain. On ceremony consumption, the receiver MUST cancel every other active pairing connection attached to that ceremony. After any required outgoing confirmation has been materialized, every owner MUST promptly best-effort zeroize and release its owned code, `digest`, `w`, local `x` or `y`, `K`, secret `TT` intermediates, `Ke`, `Ka`, `KcA`, `KcB`, and expected confirmation tags. The same cleanup applies to each owner on failure, timeout, disconnect, expiry, exhaustion, or cancellation, as soon as the value is no longer needed.

## Failures and attempts

A failed attempt is charged only when a syntactically valid, correctly ordered sender confirmation is admitted to verification against an active ceremony and that verification fails. The count is not a count of connections, shares, PAKE computations, malformed frames, invalid points, wrong lengths, wrong-state messages, or premature disconnects. Those inputs do not consume an attempt, but finite in-flight pairing-state limits, frame limits, deadlines, per-source controls, and global connection and CPU rate limits MUST prevent unlimited expensive work.

All failed admitted sender-confirmation verifications caused by a wrong code, exporter or `hello` binding mismatch, or otherwise valid but incorrect sender tag share one terminal result: `error` with `code: "authentication_failed"` when safe, or a silent close. An unavailable, expired, exhausted, consumed, or nonexistent ceremony MAY fail earlier at any pairing phase with that same code or a silent close. Implementations MUST NOT perform a dummy PAKE merely to conceal ceremony state. Phase, timing, resource use, and ceremony availability are explicitly not hidden, and this protocol does not claim that ceremony existence or expiry is indistinguishable. No response reports attempts remaining, and there is no in-connection retry. A new online guess requires a new TLS connection and is charged only if it is admitted to sender-confirmation verification and fails.

## Consent boundary

Pairing completes before `transfer_request`, so filenames and sizes are not disclosed to an unauthenticated endpoint. After mutual pairing, the sender sends the manifest, the receiver separately reviews and approves it, and authenticated TLS protects `transfer_request`, `transfer_response`, `ready`, and DATA.

The PAKE does not bind a manifest or approval that did not yet exist. Pairing authenticates the direct TLS connection and peer knowledge of the out-of-band code; receiver consent is a later state transition protected by that authenticated TLS connection.

## Rust implementation gate

The intended transport mapping is the `rustls` 0.23 family with `tokio-rustls` 0.26, `rcgen` 0.14, and `zeroize`, plus a small pairing adapter. No existing Rust SPAKE2 crate is approved for production use.

`pakery-spake2` 0.2.1 is only a prototype candidate because it is new and unaudited. RustCrypto `spake2` is explicitly unsuitable: it implements an old draft, is unaudited, and is probably not constant-time. Production requires an audited RFC 9382-conformant dependency, RFC 9382 vectors for this ciphersuite, tests for every wire encoding and state transition, and specialist review of exporter composition, certificate verification, scalar mapping, constant-time behavior, concurrency accounting, and best-effort secret zeroization. Whether ephemeral private-key buffers created or retained by `rcgen` and `rustls` can be reliably zeroized remains an explicit release-review gate; the specification does not claim guaranteed erasure of those buffers.

See [Protocol](PROTOCOL.md), [Message Format](MESSAGE_FORMAT.md), [Transport](TRANSPORT.md), [Security](SECURITY.md), and [Threat Model](THREAT_MODEL.md).
