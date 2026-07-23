# Versioning

## Version Types

- **Application version** identifies a Lanweave build.
- **Specification revision** identifies a documentation edition.
- **Protocol version** identifies the complete wire messages and lifecycle.
- **Frame version** identifies the binary frame header.

The draft MVP protocol and frame versions are both `1`, but these are separate values.

## Draft Reset

No protocol implementation has shipped. The version 1 draft was reset in July 2026 to define the TUI-driven pairing request and reusable session model. Older documentation that described receiver-started pairing and one transfer per connection is not a supported protocol version.

After the first interoperable implementation is published, incompatible changes require a new protocol version.

## Version 1 Order

1. TLS 1.3 with ALPN `lanweave/1`.
2. Initiator and responder `hello` messages.
3. `pair_request` and `pair_response`.
4. Local code display and entry after acceptance.
5. Four pairing records and mutual confirmation.
6. Zero or more separately approved transfer exchanges.
7. Manual close, idle close, shutdown, or failure.

Sending file metadata before pairing, sending data before approval and `ready`, or reusing authorization on another connection is a protocol error.

## Compatibility

Version 1 rejects unknown fields, messages, enum values, frame kinds, and wrong-state transitions. An incompatible change includes:

- adding or changing a field, message, reason, code, or frame kind;
- changing limits, ordering, roles, consent, or failure scope;
- changing TLS, pairing, exporter, identity, or code behavior; or
- adding trusted devices, resume, negotiation, or simultaneous transfers.

Discovery's version value is only an untrusted hint. Peers confirm exact version `1` in `hello` and do not automatically fall back to another protocol.
