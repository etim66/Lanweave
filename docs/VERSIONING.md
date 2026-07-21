# Versioning

Lanweave has three kinds of version number, and I keep them separate so a documentation edit is not mistaken for a wire-protocol break.

## Three version domains

- **Specification revision 0.1** identifies this document set and may change while the protocol is draft.
- **Protocol version 1.0** is the proposed first wire contract described by revision 0.1. It is not implemented yet.
- **Application version** identifies a build of `lanweave` or another implementation and does not determine wire compatibility.

Framing has a separate one-byte format version (proposed `1`) so parsers can reject incompatible headers before deserialization.

## Compatibility rules

1. Different protocol major versions are incompatible on one connection.
2. Same-major peers choose the highest minor version supported by both, subject to local policy.
3. Both complete negotiation before transfer messages.
4. Optional behavior uses explicit capabilities; minor version alone never implies use without negotiation.
5. Unknown optional fields may be ignored only if the codec/schema marks them non-critical and their omission cannot change security semantics.
6. Unknown critical fields, required capabilities, or message types cause `UnsupportedVersion` or `InvalidMessage` and close.
7. Unknown enum values are never coerced to a known value. Extensible non-critical enums preserve/ignore unknown values as their schema defines; closed security/state enums reject them.
8. Offered ranges, selected version, capabilities, framing, transport, and algorithm suite are bound to the authenticated transcript to prevent downgrade.

## Negotiation

Each peer states the protocol ranges, frame versions, algorithms, profiles, and capabilities it supports. The responder selects a compatible set in `ProtocolNegotiationAccepted`, and the initiator checks that every selected value was actually offered.

Before either side processes a transfer request, both must have the same transcript view. This is not bookkeeping: including the offers and selection in the authenticated transcript is what stops an intermediary from quietly choosing weaker settings.

Capabilities have stable ASCII identifiers or numeric registry values, a critical/optional bit, and parameters with explicit bounds. Required capabilities must be understood and selected. An advertisement capability is only a hint; connection negotiation is authoritative.

## Schema evolution

- Add a non-critical optional field with a defined default in a minor version.
- Never reinterpret an existing field or enum value.
- Add a new optional message only behind a negotiated capability.
- Adding a mandatory phase/field/message generally requires a new major version.
- Deprecate by first making send behavior optional while receivers continue parsing; remove only in a major version.
- Preserve canonical field ordering/encoding rules used in signatures/transcripts. Unknown fields cannot be blindly reserialized into a signed transcript; transcript definition must specify raw/canonical inclusion.
- Never use “field absent” to silently select a weaker security behavior.

## Examples

Assume an implementation labeled 1.x supports all minors from 1.0 through its maximum unless it advertises gaps.

| Peer A | Peer B | Result |
| --- | --- | --- |
| 1.0 | 1.1 | Negotiate 1.0; 1.1-only optional features remain off |
| 1.0 | 1.5 | Negotiate 1.0; B must retain 1.0 semantics and cannot send unknown critical additions |
| 1.0 | 2.0 | Reject: no common major; no transfer request processed |

A strict implementation may withdraw an insecure old minor. It then advertises a minimum above it and fails closed rather than accepting a downgrade for compatibility.

## Framing versions

The magic identifies Lanweave; frame-format version identifies header semantics. Unknown frame version is rejected before trusting payload length beyond the minimal fixed header. A future transport with native message/stream framing may still define an equivalent profile and negotiate it; it cannot silently reuse frame version 1 with different semantics.

## Registries and governance

Before the wire format is frozen, I need small registries for message types, error codes, flags, capabilities, algorithms, and critical fields. Experimental values should live in ranges that production peers reject unless they have been explicitly enabled.

Every specification change that touches those registries should say what remains compatible and add or update the corresponding test vectors.
