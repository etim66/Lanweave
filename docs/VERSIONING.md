# Versioning

## Separate version domains

Lanweave keeps these identifiers distinct:

- **Application version** identifies an implementation build. It is diagnostic and never establishes wire compatibility.
- **Specification revision** identifies an edition of the documentation. Editorial changes may update it without changing the wire protocol.
- **Protocol version** identifies the complete message schema and state behavior. The experimental MVP protocol version is the integer `1`.
- **Frame-format version** identifies the 12-byte header semantics. The MVP frame-format version is the integer `1`.

Equal numbers in different domains do not make them interchangeable. Discovery may advertise a protocol-version hint, but discovery is untrusted and the hint neither selects a version nor authorizes fallback.

## Exact version 1 behavior

The sender sends `hello` under TLS before any other Lanweave control message. The receiver validates it and sends exactly one `hello` response. Each `hello` version MUST be exactly `1`. There are no major/minor ranges and no version negotiation.

The following order is normative for protocol version 1:

1. Establish TLS 1.3 with exact ALPN `lanweave/1`, a fresh per-connection self-signed ECDSA P-256 certificate, no resumption, and no early data. Provisional certificate acceptance relaxes only trust-anchor and name validation; strict certificate structure and signature checks, including `CertificateVerify`, remain mandatory.
2. Exchange the two exact version 1 `hello` messages, each with a JSON body of at most 4,000 bytes excluding its frame header.
3. Complete the fixed SPAKE2 pairing profile and mutual key confirmation. The live TLS exporter and the two exact `hello` JSON body byte strings in wire order, excluding frame headers, are RFC 9382 AAD and are not part of transcript `TT`; the fixed maxima keep that AAD within the RFC bound.
4. Only after pairing, send `transfer_request` with the complete manifest.
5. Obtain separate receiver approval and prepare the destination. Preparation failure sends a rejecting `transfer_response`; success sends an accepting `transfer_response` followed by `ready`. Pairing is not transfer consent.
6. Send DATA and file controls only after `ready`.

Sending a manifest or any filename, size, or file count before pairing, approving a transfer before its manifest is displayed, or sending DATA before `ready` is a wrong-state protocol error.

If a valid frame contains a bounded, safely parseable `hello` with a different protocol version, the receiver sends `error` with `unsupported_version` and closes. If framing, length, UTF-8, JSON, or enough of the schema is invalid to make that response unsafe, it closes without an error. A peer MUST NOT retry a lower version on the same connection.

Frame-format version 1 is also exact. A parser validates the frame version in the header before interpreting its payload. An unknown frame version causes immediate close because its payload length and control schema cannot be assumed safe enough for an error response.

Version 1 has exactly one transport profile, frame format, control codec, and pairing profile: TLS 1.3, ALPN `lanweave/1`, ephemeral per-connection self-signed ECDSA P-256 certificates, RFC 9382 SPAKE2-P256-SHA256 with sender A and receiver B, HKDF-SHA256, HMAC-SHA256 mutual confirmation, and TLS-exporter plus exact-`hello`-body AAD binding. TLS is the sole record-protection layer after binding; the PAKE key is not a file or application traffic key. Version 1 has no negotiation of Lanweave capabilities, version ranges, PAKE algorithms or profiles, frame kinds or versions, transport, application AEAD, optional messages, or extensions. This does not disable standard TLS 1.3 cipher-suite and key-exchange negotiation, which occurs only inside one pinned and reviewed TLS provider/version/configuration policy.

This fixed selection remains experimental, unaudited, and release-blocked for production assurance pending specialist mapping/composition, provisional-certificate verifier, and pinned TLS provider policy review and approval of an audited RFC-conformant Rust dependency. No currently evaluated crate is approved.

## Strict schema compatibility

Version 1 rejects every unknown field, message type, enum value, frame kind, and state transition. It does not ignore optional-looking additions from a newer peer. Field absence is valid only where the version 1 schema explicitly defines that field as optional and specifies its semantics.

Every control payload at or below its fixed version 1 parser maximum MUST be read and parsed under this schema. Lower implementation policies MUST NOT alter pre-pairing parsing or the wire maxima; lower file-count, total-size, storage, or resource policies are evaluated only after pairing as manifest/storage admission and result in a rejecting `transfer_response`.

Any incompatible schema or framing behavior requires a new protocol or frame-format version as applicable. This includes:

- adding, removing, or reinterpreting a field, message, enum value, frame kind, or lifecycle phase;
- changing JSON types, integer bounds, requiredness, defaults, limits, or validation;
- changing message ordering, roles, consent semantics, manifest binding, or failure behavior;
- adding a Lanweave capability or PAKE algorithm/profile choice;
- changing frame-header semantics or interpreting a DATA payload differently;
- changing the TLS profile or pinned provider policy, certificate lifecycle or verifier requirements, ALPN, SPAKE2 group/hash/roles, password ceremony, KDF, confirmation, exporter, AAD or transcript binding, failed-confirmation accounting, or record-protection composition.

Even an additive field requires a new protocol version because version 1 receivers reject unknown fields. Any pairing or security-profile change requires a new protocol version, even if its wire shape appears compatible. A new version must define a complete fixed profile rather than silently extending version 1.

## Future versions

A future implementation may support more than one exact protocol version, but selection and downgrade resistance require a separately specified design. It must not retrofit ranges or capability negotiation into version 1. Failed version 1 connections do not authorize automatic insecure fallback; a new attempt is a new connection governed by explicit local policy and that future version's reviewed rules.

QUIC, custom AEAD, algorithm agility, persistent trusted devices, and resume are deferred features, not dormant version 1 capabilities.

See [Transport](TRANSPORT.md), [Message Format](MESSAGE_FORMAT.md), and [Cryptography](CRYPTOGRAPHY.md).
