# Discovery

## Purpose And Trust

Lanweave uses mDNS/DNS-SD as an optional convenience for finding candidate endpoints on the local link. An advertisement answers only “where might a version 1 Lanweave listener be?” It does not prove identity, consent, pairing, availability, or safety.

Discovery is not part of the active transfer state. Once a TCP/TLS connection is selected, an advertisement changing, expiring, or disappearing has no effect on that connection or its transfer.

## Service Record

The proposed service type is:

```text
_lanweave._tcp.local.
```

SRV supplies the host and port. TXT contains at most this hint:

| Key | Value | Meaning |
| --- | --- | --- |
| `v` | `1` | Untrusted hint that the listener expects experimental protocol version `1` |

There are no discovery capabilities, identity fingerprints, platform values, application versions, transfer state, file metadata, display names, pairing codes, or pairing-ceremony state in TXT. Pairing codes and ceremony state MUST NOT appear in any PTR, SRV, TXT, service-instance label, host label, or other discovery field. Service-instance and host labels remain untrusted and must be safely rendered.

## Behavior

1. Start the TCP listener before advertising it.
2. Register `_lanweave._tcp.local.` and browse while discovery is enabled.
3. Resolve PTR, SRV, optional TXT, and A/AAAA records into bounded candidate endpoints.
4. Preserve interface scope for link-local IPv6 addresses and retain both IPv4 and IPv6 candidates.
5. Treat duplicates, conflicts, updates, stale records, and goodbye records as ordinary discovery events.
6. Stop discovery independently of any active connection.

Implementations must bound record counts, text lengths, retained candidates, resolution work, and connection attempts. Record values are hostile input. Addresses are routing information, not peer identity.

## Direct-Address Fallback

A user-supplied host or IP address and port bypasses mDNS but not provisional TLS, `hello`, pairing, or any security check. Direct-address connections MUST pair first, then send `transfer_request`, then obtain separate exact-manifest approval and prepare the destination before an accepting `transfer_response`; acceptance is followed by `ready`. They MUST NOT disclose names, sizes, or file counts before pairing. Provisional certificate acceptance still requires strict certificate structure and signature validation, including TLS 1.3 `CertificateVerify`. This fallback is required because multicast may be blocked by firewalls, guest networks, VPNs, client isolation, or platform policy.

## Failure And Security

Spoofed discovery can hide a peer, create noise, or direct a connection to an attacker. The fixed experimental, unaudited, release-blocked pairing profile is intended to prevent that provisional endpoint from becoming the paired peer without the receiver-generated out-of-band string; it is not yet a production assurance claim. Because pairing precedes `transfer_request`, a spoofed provisional endpoint does not receive manifest names or sizes. Discovery code must report "not found," resolution failure, connection failure, and protocol/pairing rejection as distinct outcomes without exposing cryptographic failure details. Within pairing, malformed messages or points use `invalid_message`, while an unavailable ceremony may fail before the receiver share and uses the same `authentication_failed` code as a valid correctly ordered sender-confirmation failure when a response is safe. This shared code does not guarantee equal phase or timing and does not claim to hide ceremony existence.

Discovery is not private. Other LAN devices can observe service presence and advertised endpoints even when they cannot authenticate as a peer; once a connection begins they can also observe timing and traffic volume, capture or drop ciphertext, and cause denial of service.

Platform prototypes must verify registration APIs, firewall behavior, multi-interface handling, scoped IPv6, conflict renaming, TTL expiry, and multicast-disabled operation. Until those tests pass, mDNS library and adapter choices remain implementation research rather than protocol commitments.
