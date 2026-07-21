# Discovery

## Role of mDNS and DNS-SD

Lanweave uses mDNS to find records on the local link and DNS-SD to describe the service behind those records. In practice, DNS-SD provides PTR, SRV, TXT, and address records without asking the user to configure a DNS server.

The service type for the initial TCP profile is:

```text
_lanweave._tcp.local.
```

An advertisement answers “where might a compatible instance be listening?” It never proves device identity, consent, availability, or safety.

## Proposed advertisement

The service instance label is there for local browsing. It should be readable and allowed to change when there is a naming collision; it is not the device's identity.

The SRV record already carries the target host and listening port. I therefore do not plan to repeat the port in TXT, even though the table records that option for completeness.

| TXT key | Proposed encoding | Status | Privacy/security analysis |
| --- | --- | --- | --- |
| `id` | Truncated/encoded device ID, or omit | Needs Research | Stable IDs enable cross-session tracking. Prefer an ephemeral discovery pseudonym and reveal persistent ID after connect. |
| `inst` | Instance ID, compact ASCII | Proposed | Enables local self-filtering/restart distinction but tracks one process lifetime. |
| `v` | Supported major and minor range, e.g. `1.0-1.0` | Proposed | Reveals protocol generation; enables filtering but remains untrusted. |
| `port` | Decimal port | Rejected duplication | SRV already carries it; conflicting values create ambiguity. |
| `name` | UTF-8 device display name | Deferred/default omit | Useful UX but can reveal a real name, host purpose, or terminal-control text. Service instance label may already expose a name. |
| `platform` | Coarse value or omit | Default omit | Helps diagnostics but fingerprints OS and attack surface. Negotiate after connection if needed. |
| `cap` | Compact non-sensitive discovery capabilities | Proposed minimal | Advertise only connection-relevant hints; detailed capabilities enable fingerprinting and remain untrusted. |
| `fp` | Encoded public-key fingerprint | Needs Research/default omit | Supports early continuity but is stable tracking metadata. Prefer disclosure on connection unless threat review favors it. |
| `app` | Application version | Default omit | Useful debugging, but advertises patch level/vulnerabilities. Protocol version is sufficient for discovery. |

TXT records have practical size and interoperability constraints. Keep the total comfortably below a single typical DNS packet, use ASCII keys, define exact encodings later, and treat all values as hostile. The normative listening port comes from SRV.

## Lifecycle

1. Generate a new random instance ID at process start.
2. Listen successfully before registering the service.
3. Register one service instance per logical listener, with conflict-renaming handled by the platform.
4. Browse for `_lanweave._tcp.local.` continuously while discovery is enabled.
5. Resolve PTR → SRV/TXT → A/AAAA and retain multiple scoped candidate addresses.
6. Filter the local instance by instance ID plus local listener/address checks; never filter solely by display name.
7. Emit added/updated/removed events to application orchestration.
8. Refresh TTLs; expire stale entries after the advertised TTL or recommended 120 seconds.
9. On graceful exit, deregister/send goodbye. Consumers must still tolerate crashes and stale caches.

## Address handling

- Keep both IPv4 and IPv6; do not convert addresses to identity.
- Preserve IPv6 scope/interface IDs for link-local addresses.
- Try candidates with bounded “happy eyeballs”-style racing or ordered fallback, avoiding connection storms.
- A device on multiple interfaces may appear multiple times. Deduplicate provisionally by instance ID, then by authenticated device identity after connection.
- Do not bridge advertisements across interfaces by default. Respect OS routing and administrator policy.
- Hostname and address changes update the candidate; they do not change authenticated identity.
- Duplicate service names are normal and automatically suffixed; UI should show enough non-sensitive context to distinguish candidates.

## Failure realities

Discovery will fail on some perfectly healthy networks. A firewall may block inbound TCP or UDP 5353, guest Wi-Fi often isolates clients, and some access points suppress multicast. VPN and container interfaces can also make the same machine appear more than once.

Those are availability problems, not authentication failures. The CLI should tell the difference between “nothing was advertised,” “the service resolved but the connection failed,” and “a peer connected but rejected the protocol.”

Future Android work must verify multicast locks, background execution, battery policy, and OEM restrictions. Mobile behavior must not be inferred from desktop success.

## Alternatives

| Mechanism | Advantages | Limitations / role |
| --- | --- | --- |
| mDNS/DNS-SD | Standard local service discovery, IPv4/IPv6, mature OS integration | Multicast/client isolation, metadata leakage, platform differences |
| SSDP | Widely seen in consumer networks | HTTP-like discovery semantics, noisy ecosystem, weaker service typing |
| Direct IP entry | Works without multicast and is simple fallback | Poor UX, changing/scoped IPv6 addresses, still needs authentication |
| UDP broadcast | Simple on IPv4 | Not IPv6-native, subnet-limited, custom discovery protocol/firewall issues |
| Wi-Fi Direct | Can work without shared AP | Complex platform-specific group creation; future transport topology |
| Bluetooth discovery | Useful proximity hint | Permissions, bandwidth, platform complexity; future assisted discovery only |
| Central rendezvous | Cross-subnet/internet reachability | Violates no-mandatory-server goal; privacy/operation burden |

For version 1.0, I plan to use mDNS/DNS-SD and keep direct address entry as a later fallback. I will advertise as little as practical and verify every useful claim again after connecting.

This remains a proposed decision until the platform tests are complete. The relevant attacks and privacy costs are covered in the [Threat Model](THREAT_MODEL.md).

## Verification questions

I still need real results for the Rust libraries, Windows firewall prompts, macOS registration, Linux daemon dependencies, interface selection, TXT record size, and scoped IPv6 addresses.

That work is scheduled in the [Research Plan](RESEARCH_PLAN.md); until it is done, the discovery decision stays proposed.
