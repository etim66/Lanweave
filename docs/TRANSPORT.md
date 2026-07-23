# Transport

## MVP profile

Protocol version 1 uses one TCP connection for at most one transfer. TLS 1.3
starts immediately after TCP establishment and before either peer sends a
Lanweave `hello`. ALPN is exactly `lanweave/1`; absence or mismatch fails the
connection.

The receiver creates a fresh self-signed ephemeral P-256 certificate and private
key for every connection, holds them only in memory, drops every owner of those
objects when the connection ends, and performs best-effort zeroization where
the `rcgen` and `rustls` private-key APIs permit. Physical or guaranteed erasure
of `rcgen`/`rustls` internal buffers is not claimed; that remains a release-review
gate. The server sends exactly that one end-entity certificate and no
intermediates. Client and server resumption caches, session tickets, PSKs,
0-RTT, and half-RTT application data are disabled. A reconnect therefore performs
a full TLS 1.3 handshake with a new receiver certificate and new TLS exporter.

Before pairing, the certificate identity is provisional: it does not
authenticate discovery claims, a stable device, user consent, or a transfer.
The sender's custom verifier bypasses only PKI trust-anchor and DNS/IP name
validation. It MUST require exactly one structurally valid DER certificate, no
intermediates, a P-256 public key, and the expected self-signed P-256 form,
including a valid certificate self-signature. DER parsing, validity, key-usage,
and profile checks are not bypassed. It MUST advertise and accept only
`ECDSA_NISTP256_SHA256` for TLS 1.3
CertificateVerify and use rustls's supported signature verification for that
scheme. Its TLS 1.2 signature-verification entry point MUST fail. Rustls MUST
validate the rest of the TLS 1.3 handshake, including Finished; neither custom
verification nor application code may synthesize handshake success.

TLS version is exactly TLS 1.3 and ALPN is exactly `lanweave/1`. Standard TLS 1.3
ciphersuite and key-exchange negotiation is allowed only within the explicitly
pinned rustls crypto provider configured by the implementation. This is TLS
handshake negotiation, not Lanweave negotiation. There is no Lanweave transport,
capability, version-range, framing, algorithm, PAKE/pairing-profile, or
application-AEAD negotiation in version 1. A connection that does not implement
the fixed profile fails closed. No Lanweave frame may be sent until the full TLS
1.3 handshake, including both Finished messages and ALPN selection, completes.

## Pairing binding

After TLS, the sender sends `hello` and the receiver replies with `hello`. Each
`hello` JSON body is at most 4,000 bytes. Each peer retains the exact sender and
receiver `hello` JSON body bytes as transmitted, including whitespace and field
order but excluding the 12-byte frame headers, rather than a re-serialization.
The fixed pairing profile obtains exporter keying material from that TLS
connection and binds the exporter and those exact body bytes, in wire order, as
SPAKE2 additional authenticated data (AAD). The adapter supplies that byte
string as RFC 9382 AAD for confirmation-key derivation; it MUST NOT insert the
AAD into the RFC transcript `TT`.

The pairing profile is fixed RFC 9382 SPAKE2 with P-256, SHA-256, HKDF, and HMAC;
the sender is Party A. HKDF derives the HMAC confirmation keys, and the
confirmations authenticate the fixed pairing transcript and AAD. Pairing has
exactly four records and no negotiation:

1. Sender share.
2. Receiver share.
3. Sender confirmation.
4. Receiver confirmation.

The receiver sends its final confirmation (`cB`) only after the sender
confirmation (`cA`) is valid for the active receiver ceremony and the current
connection's exporter, hello bodies, and SPAKE2 transcript. Valid `cA`
atomically changes the ceremony to `consumed_sender_confirmed`; this is not yet
pairing success. The receiver connection becomes paired only when its sole TLS
writer accepts the complete framed `cB` in protocol order and flush succeeds.
Here committed means writer acceptance plus that successful flush. It is not a
delivery guarantee: the sender becomes paired only after receiving and
verifying complete `cB`. Pairing authenticates only this TLS connection. The
ceremony code is not sent on the connection.

If the ceremony is expired, exhausted, cancelled, already sender-confirmed, or
otherwise unavailable when a valid sender share arrives, the receiver performs
no dummy PAKE. It may send the same coarse `authentication_failed` used for
later valid confirmation failures, or silently close. The error code is coarse;
the earlier phase, timing, and ceremony availability are explicitly not hidden.

TLS is the sole record-protection layer for pairing controls, transfer controls,
and DATA. SPAKE2 supplies password-authenticated binding and key confirmation;
version 1 does not wrap records in a second application AEAD or replace TLS
record protection with PAKE-derived encryption.

## Preferred Rust Mapping

The preferred initial implementation pin is `rustls` 0.23 with the AWS-LC
cryptographic provider, `tokio-rustls` 0.26, `rcgen` 0.14, and `zeroize`. The
implementation must:

- construct the provider explicitly and disable default TLS 1.2 support,
  resumption caches, session tickets, PSKs, 0-RTT, and half-RTT application
  data;
- require exactly TLS 1.3 and the fixed ALPN `lanweave/1`;
- verify that `HandshakeKind::Full` followed each TLS handshake, not a
  resumption or externally provided PSK; and
- obtain the RFC 9266 exporter only after handshake completion.

Release builds must disable key-logging facilities such as the `log` feature in
`tokio-rustls` and any provider or environment variable for key material.

The sender needs a custom provisional server-certificate verifier. Its narrow
responsibility is to bypass trust and name checks for the expected fresh
self-signed certificate without claiming that it is a stable identity. The
certificate callback still validates exact chain cardinality, DER structure,
validity and key usage when present, P-256 key/profile, and the self-signed
certificate signature. The verifier exposes only `ECDSA_NISTP256_SHA256`,
delegates TLS 1.3 `CertificateVerify` validation to `rustls`'s supported helper
for that scheme, and makes its TLS 1.2 verification callback fail. It must not
return unconditional success for handshake signatures. The selected rustls
provider and its allowed TLS 1.3 ciphersuites and key-exchange groups are pinned
rather than controlled by peer-provided Lanweave fields.

No existing Rust PAKE crate is approved for this profile pending focused API,
constant-time, dependency, maintenance, RFC 9382 conformance, and test-vector
audit. The candidate prototype surface is `pakery-spake2` 0.2.1 together with
`pakery-core` 0.2.1 and `pakery-crypto` 0.2.1 using its `p256` and `spake2`
features; review covers the entire feature-selected dependency graph, and the
adapter discards `Ke`/`session_key` after confirmation. RustCrypto `spake2` is
explicitly unsuitable. The crate choice must not change the fixed wire profile.

## Stream and framing

TLS carries one reliable, ordered byte stream. Both control messages and file bytes use the existing 12-byte frame header defined in [Message Format](MESSAGE_FORMAT.md):

- A control frame payload is exactly one UTF-8 JSON object.
- A DATA frame payload is raw file bytes, not JSON and not a JSON envelope.
- The frame kind determines how the payload is interpreted. Unsupported frame kinds are errors.
- Frame-format version 1 and fixed version 1 limits apply; limits are not negotiated.

The receiver accumulates partial headers and payloads and handles multiple
frames in one socket read. It MUST validate magic, frame version, header length,
frame kind, payload length, and the generic kind maximum before allocating or
reading the payload. For JSON, the preallocation ceiling available from the
header alone is 1 MiB; the implementation MUST NOT preallocate the peer's
claimed size above that ceiling. It invokes the JSON parser only after the
complete, generically length-validated body is available, then applies the
message-type and current-state limit, including 4,000 bytes for each `hello`,
before acting on the message. Type-specific limits are not assumed from
unparsed bytes. DATA uses its fixed kind-specific limit before allocation.
Invalid framing closes the connection. After bounded parsing, an oversized
`hello` or `pairing` is `invalid_message` when safe; a structurally valid
oversized `transfer_request` receives a rejecting `transfer_response` with
`invalid_manifest`.

One writer serializes frames onto the stream. Queues and file reads are bounded so socket backpressure reaches the source. DATA and sequencing controls such as `file_end` retain FIFO order. A terminal `cancel` or `error` may bypass the queue only after all queued nonterminal frames have been atomically discarded; bytes already written cannot be reordered or interrupted.

## Connection lifecycle

The successful version 1 lifecycle is:

1. Sender `hello`, then receiver `hello`, each containing exact protocol version 1.
2. Four `pairing` records in exact order: sender share, receiver share, sender confirmation, receiver confirmation.
3. `transfer_request` from the paired sender with the complete, immutable manifest.
4. Separate receiver validation and local whole-manifest consent.
5. Destination preparation, then `transfer_response` accepting that exact manifest.
6. `ready` from the receiver after the accepting response; no preparation remains pending, but DATA is still forbidden until this message.
7. Zero or more raw DATA frames for the current file, followed by `file_end` from the sender.
8. `file_result` from the receiver. A successful result advances to the next manifest entry.
9. The final verified `file_result` completes the successful transfer, after which the connection closes.

A manifest or user rejection closes after the rejecting `transfer_response`.
Destination-preparation failure also sends a rejecting `transfer_response`; an
accepting response is never sent before preparation succeeds. During a file,
the receiver may send a terminal failed `file_result` as soon as write, storage,
size, or destination handling fails, including before `file_end`. `cancel`,
`error`, or transport failure can terminate any applicable nonterminal phase.

No manifest or DATA is valid before pairing success. Pairing does not imply
manifest consent. No DATA is valid until the later manifest acceptance and
`ready`. A pre-pairing `transfer_request` or wrong-state DATA is
`invalid_message` when a safe response is possible.

`cancel` means cancellation of the connection and may terminate any nonterminal
phase when safely parseable. `source_unavailable` is sender-only and valid only
after an accepting `transfer_response`; other valid cancellation codes may be
used earlier. `error` is state-dependent and terminal. A peer never starts a
second transfer on the same connection. Version 1 has no session-close exchange
and no ping/pong messages. EOF, a TLS close, or a TCP close is never evidence
that a file succeeded. TLS resumption cannot carry pairing state across
connections.

Exact control-message fields belong in [Message Format](MESSAGE_FORMAT.md), while transfer and filesystem behavior belongs in [File Transfer](FILE_TRANSFER.md). Those schemas MUST conform to the fixed message set and lifecycle above; obsolete negotiation, session, chunk-envelope, acknowledgement, or completion messages are not part of this MVP.

## DATA state

Files are transferred sequentially in immutable manifest order. The ordered stream supplies byte order, so DATA frames have no application offset, sequence number, nonce, or file identifier. State identifies the one expected current file.

A DATA frame is valid only after `ready`, while receiving that current file, and only when its payload length is no greater than the file's remaining declared size. DATA in any other state, DATA after all declared bytes, or `file_end` before the exact size is received is fatal. The receiver hashes and writes only bytes that pass these checks.

After `file_end`, the receiver verifies the declared size and digest before publishing the file and sending `file_result`. On the first failure, both peers stop the transfer. The receiver deletes the current temporary file and retains only the already verified and finalized prefix of the manifest.

## Deferred scope

QUIC, multiple streams or connections, application AEAD, algorithm agility, persistent trusted devices, and resume are deferred. Adding any of them requires a separately reviewed and versioned protocol design; they are not latent version 1 capabilities.

See [Security](SECURITY.md), [Versioning](VERSIONING.md), and [Threat Model](THREAT_MODEL.md).
