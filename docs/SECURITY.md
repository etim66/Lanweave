# Security

> **Experimental:** Protocol version 1 selects the target profile below, but Lanweave has not been audited and MUST NOT be described as secure or production-ready. A security-stable release remains blocked on specialist review of the SPAKE2 mapping, TLS/exporter composition, provisional-certificate verifier, and pinned TLS provider policy and on an audited, RFC-conformant Rust dependency. No currently evaluated crate is approved.

## Trust boundary

The LAN, discovery records, addresses, provisional TLS peer, `hello`, and all protocol input are hostile. Endpoint compromise is out of scope. TLS 1.3 starts before `hello` and provides provisional confidentiality and integrity, but discovery and peer identity remain untrusted until the fixed version 1 pairing profile succeeds.

Version 1 uses TLS 1.3 with exact ALPN `lanweave/1`, a freshly generated in-memory self-signed ECDSA P-256 certificate and private key for each connection, and no TLS resumption or early data. The certificate is provisional and MUST NOT be reused as a stable identity. Provisional acceptance relaxes only trust-anchor and server-name validation: the sender MUST strictly validate the required certificate structure and signatures, including TLS 1.3 `CertificateVerify` proof of private-key possession, and reject malformed or nonconforming certificates and invalid signatures. This verifier is release-blocked on specialist review.

TLS uses its standard TLS 1.3 cipher-suite and key-exchange negotiation, constrained to one pinned and reviewed provider/version/configuration policy. The prohibition on algorithm negotiation applies to Lanweave capabilities and the fixed PAKE profile, not to standard TLS 1.3 negotiation within that policy. The selected pairing target is RFC 9382 SPAKE2-P256-SHA256 with the sender as role A and receiver as role B, HKDF-SHA256 key derivation, and HMAC-SHA256 mutual key confirmation. Its AAD contains the live TLS exporter and both exact `hello` JSON body byte strings in wire order, excluding their 12-byte frame headers; those bytes are AAD and are not inserted into RFC 9382 transcript `TT`. An exporter mismatch, including one caused by a split-TLS intermediary, MUST make confirmation fail and close the connection.

Pairing is completed before the sender sends `transfer_request`. No filename, file size, file count, manifest, or other transfer metadata may be sent before successful mutual key confirmation, so a spoofed provisional endpoint cannot learn it. Pairing authenticates and binds the live channel; it does not approve a transfer the receiver has not yet seen. The receiver MUST separately review the complete manifest and send `transfer_response` before `ready` or DATA.

After channel binding, TLS remains the sole record-protection layer. The SPAKE2-derived key is used only for the profile's derivation and mutual key confirmation; it is not a file key, application traffic key, or substitute TLS record layer. Application messages add no AEAD, nonces, record sequences, or replay window.

## Pairing ceremony

The receiver generates a uniformly random eight-digit decimal string, including possible leading zeroes, with the OS CSPRNG and shares it privately out of band with the sender. The sender enters that string as the SPAKE2 password. It MUST NOT be advertised, sent as a plain protocol field, logged, persisted, or reused.

The ceremony expires 120 seconds after generation, succeeds at most once, and permits at most five failed sender-confirmation verifications total across all connections and reconnects. A failure is counted only when a syntactically valid, correctly ordered exchange reaches sender-confirmation verification and that verification fails. Connection admission, TLS handshakes, malformed input, PAKE share processing, and other cryptographic work do not consume this five-failure count and MUST be separately bounded by concurrency, deadline, per-source, CPU, and global work limits. All ceremony data is memory-only. Verification reservation/accounting, successful consumption, and invalidation after the fifth counted failure MUST be transitions in one atomic receiver-side state machine so concurrent or reconnected confirmations cannot create a sixth verification opportunity or consume the same ceremony twice. A restart loses the ceremony rather than restoring it. On success, expiry, cancellation, or exhaustion, implementations MUST perform best-effort in-process zeroization of owned ceremony and connection-local secret buffers where the selected APIs permit and MUST promptly release all other secret-bearing objects. Language, compiler, dependency, allocator, crash, and OS copies cannot be guaranteed erased; inability to demonstrate best-effort handling of TLS private-key buffers remains a release-review gate.

The selected Lanweave/PAKE primitive suite and roles are fixed, not negotiated. Standard TLS 1.3 cipher-suite and key-exchange negotiation remains enabled only within the pinned provider policy above. This selection is not an assurance claim: exact RFC 9382 element/scalar mapping, transcript encoding, AAD construction, exporter invocation, HKDF labels, confirmation inputs, malformed-point handling, provisional-certificate validation, and composition still require specialist review and deterministic test vectors. Production implementation is blocked until that review is complete and an audited RFC-conformant Rust dependency is approved; no currently evaluated crate is approved.

## Strict control parsing

Every control payload is exactly one JSON object with these version 1 rules:

- Input MUST be valid UTF-8. The decoder rejects malformed escapes, invalid scalar handling, trailing non-whitespace data, and any representation disallowed by the field schema.
- Duplicate object member names are rejected at every nesting level. Duplicate detection MUST happen before conversion into a map that could discard them.
- Unknown fields, wrong JSON types, unknown message types, unknown enum values, and fields invalid for that message or state are rejected. Version 1 has no ignorable extensions.
- Floating-point values, including exponent or decimal notation, are forbidden. JSON integers must be in the inclusive range `0` through `2^53-1` and must also satisfy each field's narrower bounds.
- String bytes, array/map counts, total members, and nesting depth have fixed version 1 maxima. Each `hello` JSON body is at most 4,000 bytes, excluding its 12-byte frame header. With both maximum-size bodies, the fixed length prefixes, domain string, and 32-byte exporter make the SPAKE2 AAD at most 8,067 bytes, within RFC 9382's 8,176-byte bound.
- Every wire control at or below its fixed parser maxima MUST be read and parsed according to the version 1 schema. An implementation MUST NOT impose lower local limits on `hello`, pairing, or other pre-pairing controls; lower local file-count, total-size, storage, or resource policies apply only after pairing, while admitting a parsed `transfer_request`, and produce a rejecting `transfer_response`.
- The complete 12-byte header and payload length are validated against the generic frame-kind maximum before payload allocation or JSON parsing. JSON message-specific limits are applied only after bounded parsing identifies the type. Parsing an incomplete or generically unvalidated-length payload is forbidden.

Network text is data, not UI markup or a log format. User interfaces and logs render locally generated labels, visibly quote peer text where practical, and escape or remove control characters, terminal escapes, and bidirectional-control abuse. Parser, stack, and filesystem details are not reflected to the peer.

## DATA validation

DATA payloads are raw bytes and are never passed to JSON decoding. A DATA frame is accepted only after pairing, separate manifest approval, and `ready`, for the one expected current file. Before writing, the receiver checks that the frame length fits the fixed DATA maximum and does not exceed that file's remaining declared bytes.

Wrong-state DATA, excess bytes, DATA arriving before the next index is permitted, premature `file_end`, size mismatch, or digest mismatch fails the transfer. Because DATA has no file selector, the receiver cannot distinguish same-sized content chosen from another sender source. TCP ordering is the application order; there are no application offsets or replay sequences to repair or reinterpret malformed input. See [Transport](TRANSPORT.md) for the lifecycle.

## Consent and manifest

Version 1 transfers one or more regular files sequentially. The sender MUST NOT send `transfer_request` until pairing has authenticated the live TLS channel. The request contains the complete manifest that the receiver reviews only after pairing. The receiver validates the manifest, obtains consent, and prepares the destination before sending an accepting `transfer_response`; preparation failure produces a rejecting response instead. An accepting response is followed by `ready` and accepts exactly that file set, order, names, and sizes. Pairing alone is never consent, and partial acceptance or later mutation is forbidden.

The sender MUST offer only regular files. The receiver cannot inspect the sender's filesystem, so it treats every entry as an untrusted name and size and rejects the entire request before acceptance if any entry:

- is absolute, contains a path separator or traversal component, is empty/dot-like, contains NUL or forbidden controls, or is invalid or reserved on the destination platform;
- duplicates another accepted name or is platform-equivalent to it after the destination's case, normalization, trailing-dot/space, reserved-name, and alias rules;
- collides or is platform-equivalent to any existing destination name.

Version 1 does not overwrite or remotely rename an existing destination. The destination root is selected locally, and remote names are single components beneath it. Implementations use directory-relative, create-new, no-follow operations where available and revalidate containment and object type at finalization.

## File integrity and failure

The sender opens only regular files and checks source identity, type, and size before sending, then performs any additional change detection available on the source platform. A detected source change is a transfer failure; snapshot guarantees are outside version 1. The receiver streams accepted bytes into an unpredictable, restrictive temporary file in the destination filesystem and computes the required digest while writing.

Only a file with exact declared length and matching digest is atomically finalized under its accepted name. Files are sequential and the transfer stops on the first rejection, cancellation, I/O error, source change, excess data, interruption, or verification failure. The receiver deletes the current temporary file and retains the already verified, finalized prefix. It never exposes an unverified partial under a final name. Cleanup failures are reported locally with paths redacted from the peer.

Received files are never automatically executed, previewed, or opened.

## Resource and oracle controls

Implementations bound concurrent connections, pending ceremonies and prompts, control bytes, DATA bytes, manifest count and total size, nesting, queues, temporary storage, and cryptographic work. Header and cheap schema/state checks precede JSON allocation, PAKE work, hashing, and disk planning. Connection, header, payload, approval, pairing, and transfer inactivity deadlines limit resource use by slow peers; global and per-source budgets limit repeated work.

Malformed JSON, base64url, lengths, messages, and invalid or non-group SPAKE2 points are protocol-invalid input and receive only `invalid_message` when safe. The receiver MAY reject an unavailable ceremony before generating or sending its Party B share; when a response is safe, it uses the same coarse `authentication_failed` code and then closes. A syntactically valid, correctly ordered exchange that reaches sender-confirmation verification but fails because of the pairing string, exporter binding, or confirmation tag also receives only `authentication_failed` when safe and counts as one failed confirmation. This common code does not promise equal failure phase, timing, or resource use and does not claim to hide whether a ceremony exists. Parse and validate bounded encodings before group operations, use constant-time library operations for secret-dependent checks, and never reflect cryptographic details. Replayed pairing records cannot succeed on another TLS connection because confirmation binds the live exporter; replay on the same ordered connection is rejected by state.

Logs MUST exclude the pairing string, SPAKE2 values and derived keys, exporter values, confirmation tags, ephemeral TLS private keys, and full profile transcripts. They redact file content, full paths, stable peer identifiers, and peer-provided text by default. Other LAN devices can still observe discovery, endpoints, connection timing, and traffic volume, capture ciphertext, drop or delay it, and cause denial of service. Interception or disclosure of the eight-digit string defeats this pairing ceremony; TLS and pairing do not hide these network metadata.

## Deferred features

QUIC, custom application AEAD, algorithm agility, persistent trusted devices, resume, parallel files, directories, compression, and automatic acceptance are outside version 1. Each requires a new threat review and an incompatible protocol version where wire behavior changes.

See [Threat Model](THREAT_MODEL.md), [Versioning](VERSIONING.md), and [File Transfer](FILE_TRANSFER.md).
