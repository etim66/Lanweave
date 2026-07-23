# Cryptography

## Status

The version 1 profile is a design target, not an approval. Lanweave has no audited implementation and must not be described as secure or production-ready.

Release is blocked until specialists review the password mapping, TLS and pairing composition, certificate verifier, test vectors, dependency tree, and implementation. Lanweave must not implement elliptic-curve group arithmetic itself.

## Fixed Pairing Profile

The current target is RFC 9382 SPAKE2-P256-SHA256-HKDF-HMAC over TLS 1.3. The pairing initiator is Party A and the pairing responder is Party B. Transfer direction does not affect these roles.

Exact identity strings are:

```text
lanweave-v1-initiator
lanweave-v1-responder
```

Lanweave does not negotiate another pairing profile on the same connection.

## Pairing Code

The responder creates the code only after the user accepts a live `pair_request`. It uses the operating system's cryptographically secure random generator to create a uniform eight-digit decimal string. Leading zeroes are kept.

The code:

- is displayed locally by the responder;
- is entered locally by the initiator;
- is never sent as a plain protocol field;
- remains only in memory;
- expires after 120 monotonic seconds;
- works for one connection and one cryptographic pairing attempt.

Anyone who sees the live code may be able to pair. Users must show it through a private channel, such as reading it directly from the other screen.

The application-specific password-to-scalar mapping remains a specialist-review item. Leading zeroes must be part of the exact password bytes.

## TLS Profile

The peers complete TLS 1.3 before Lanweave messages. The responder uses a fresh self-signed ECDSA P-256 certificate for each connection. The initiator validates the required certificate and handshake signatures but does not treat the certificate as a stable identity.

ALPN is `lanweave/1`. TLS 1.2, resumption, tickets, 0-RTT, and early data are disabled. Each connection therefore has fresh TLS state and a fresh exporter.

## Binding Pairing To TLS

Each peer derives a 32-byte TLS exporter after handshake completion. Pairing confirmation includes additional authenticated data made from:

```text
fixed profile label
TLS exporter
exact initiator hello JSON body
exact responder hello JSON body
```

Each field is length-prefixed. The exact JSON bytes are used without reformatting. Each `hello` body is limited to 4,000 bytes.

This binding is intended to stop an intermediary from completing pairing across two different TLS connections. The composition requires specialist review and adversarial tests before release.

## Exchange

After `pair_response` accepts and the initiator enters the code, the peers exchange:

1. initiator Party A share;
2. responder Party B share;
3. initiator Party A confirmation; and
4. responder Party B confirmation.

Both confirmations are required. The responder consumes the code when the initiator confirmation succeeds. The initiator succeeds only after verifying the responder confirmation. A lost final confirmation fails safely and does not make the code reusable.

TLS remains the only layer that encrypts application traffic. Pairing-derived values are not file keys and are not used to add another record-encryption layer.

## Cleanup And Attempts

Code-derived values, ephemeral pairing values, confirmation keys, and copies of exporter material receive best-effort zeroization as soon as they are no longer needed. This cannot guarantee physical erasure from every compiler, allocator, library, crash dump, or operating-system copy.

An accepted request allows one cryptographic pairing attempt. Any confirmation failure closes the connection and destroys the code. Malformed frames and shares also close that connection. Separate connection, CPU, memory, prompt, and rate limits must bound abusive work.

Authentication failures use one coarse `authentication_failed` result when safe, or a silent close. They do not report the code or exact cryptographic cause.

## No Trust Persistence

Pairing authorizes only the current connection. Lanweave stores no trusted-device record, reusable certificate, pairing secret, or reconnect token. Closing the session destroys authorization and the next session repeats pairing.

## Implementation Gate

No Rust SPAKE2 crate is currently approved. Any candidate must be checked for RFC 9382 conformance, maintenance, constant-time behavior, audit status, dependency risk, API fit, and secret handling. Prototype results do not replace an independent audit or specialist review.
