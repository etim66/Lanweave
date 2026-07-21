# Transport

## Recommendation

I plan to start with **TCP and TLS 1.3**. One reliable ordered stream is enough for the first single-file transfer, and TCP is easier to deploy and debug on the desktop platforms I care about. QUIC remains an option, but I will only adopt it in response to a measured need, not simply because it is newer.

That recommendation is conditional on solving the pairing problem. A TLS connection to a peer whose identity has not yet been verified may hide traffic from a passive observer, but Lanweave does not call it a **secure session**. The protocol only uses that term after consent, token verification, device identities, negotiation, and the key exchange have been bound together and authenticated.

The exact construction is still open and needs review. [Cryptography](CRYPTOGRAPHY.md) explains the boundary I need to preserve.

## TCP/TLS 1.3 versus QUIC/TLS 1.3

| Concern | TCP + TLS 1.3 | QUIC + TLS 1.3 |
| --- | --- | --- |
| Complexity | Familiar stream, separate framing/backpressure | More concepts: connection IDs, streams, transport parameters, UDP tuning |
| Handshake latency | TCP plus TLS; local RTT makes cost modest | Integrates transport/TLS; can reduce RTT, but 0-RTT is unsafe for non-idempotent requests unless disabled |
| Encryption | TLS 1.3 via a library | Mandatory TLS 1.3 integrated with QUIC |
| Multiplexing | One ordered stream unless extra connections | Independent streams built in |
| Head-of-line blocking | Lost packet stalls later bytes in connection | Loss on one stream need not block another stream |
| Migration | TCP connection breaks on address/interface change | QUIC may migrate if library/platform/network permits |
| File performance | Strong for one stream on LAN; mature kernel tuning | Strong potential for parallel files; extra UDP/user-space behavior |
| Rust maturity | Tokio + rustls ecosystem is established; exact current APIs must be verified | Quinn is prominent; current API/platform maturity must be verified |
| Firewalls | TCP listener is familiar, often easier | UDP may be filtered/rate-limited; unfamiliar prompts/policies |
| Debugging | Standard socket/TLS tooling; decrypted app logs still needed | Encrypted UDP and QUIC internals are harder to inspect |
| Resume support | Application checkpoints/reconnect required | Streams do not automatically create durable resume |
| Mobile future | Broad platform support, but interface changes break sessions | Migration and streams are attractive, with more platform/UDP constraints |

Neither transport solves consent, identity, safe paths, file hashes, durable resume, or application state.

## Physical message path

1. mDNS resolves a peer's IP address(es) and SRV listening port.
2. The initiator opens a transport connection with bounded candidate attempts.
3. The peers negotiate protocol compatibility. If TLS starts immediately, negotiation messages are channel-encrypted but not yet pairing-authenticated.
4. The sender constructs a typed protocol message.
5. The selected wire codec serializes it into bounded bytes.
6. Framing adds magic/version/flags/length.
7. The complete frame is written to the TCP/TLS byte stream (or a future QUIC stream).
8. The receiver incrementally reads arbitrary byte fragments.
9. It buffers only enough to reconstruct a validated header and bounded frame.
10. It authenticates/decrypts at the applicable channel/session layer and deserializes the message.
11. It validates version, message fields, IDs, role, sequence, protection requirement, and current state.
12. Validation emits a typed state-machine event.
13. Application orchestration decides the permitted response and side effects.

Reads may contain half a header, half a payload, exactly one frame, or many frames. Writes may also complete partially. Implementations loop correctly and apply backpressure rather than assuming socket-call boundaries.

## Concepts that must remain separate

- **Discovery** locates untrusted candidate endpoints.
- **Transport** moves an ordered byte stream or QUIC streams.
- **Serialization** maps typed values to/from bytes.
- **Framing** identifies message boundaries and maximum lengths on a stream.
- **Encryption/authentication** protects content and binds it to session keys/context.
- **Protocol messages** express negotiation, approval, transfer, and errors.
- **Application behavior** prompts users, chooses files/destinations, and applies policy.

## Initial connection profile

- One connection per active pairing/transfer is simplest.
- Disable TLS 0-RTT for transfer requests, token actions, and state-changing messages.
- Negotiate ALPN such as a future registered `lanweave/1` value before finalizing the profile.
- Set TCP keepalive only as a supplement to protocol inactivity timers.
- Use bounded write queues; pause file reads when the queue/window is full.
- Do not treat a TCP close as `TransferComplete`.
- Record remote endpoint changes only as diagnostics, not identity.

### Unresolved TLS start and identity model

I see three credible directions, and I need to prototype and review them before choosing one:

1. TLS immediately with self-signed/raw device identity, then token proof binds the TLS exporter and transcript (**recommended research direction**).
2. A tightly bounded plaintext preface followed by TLS after token submission (exposes token/metadata and resembles fragile STARTTLS; not recommended without proof of safety).
3. A reviewed PAKE-authenticated channel which then protects all identity/session messages (best token-guessing properties, but library/interoperability maturity must be evaluated).

What I will not do is invent a signed-X25519 handshake and treat it as finished. `KeyExchangeInit` and `KeyExchangeResponse` are placeholders for the logical steps Lanweave needs. Their final payloads cannot be frozen until the reviewed profile is chosen.

## Future stream mapping

The TCP 1.0 profile will carry control messages and file data on the same protected connection. The writer must prioritize control frames and bound its data queue so a cancellation request is not trapped behind a large amount of file data.

If I add QUIC later, one control stream and one stream per file is a sensible mapping to investigate. Separate TCP data connections add little value to the first milestone and make authentication, correlation, and firewall behavior harder.

See [Message Format](MESSAGE_FORMAT.md) for frame layout and [Design Decisions](DESIGN_DECISIONS.md) for status.
