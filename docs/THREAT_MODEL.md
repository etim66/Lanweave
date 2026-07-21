# Threat Model

## Method and scope

This STRIDE-informed model considers spoofing, tampering, repudiation/audit ambiguity, information disclosure, denial of service, and elevation through filesystem effects. Assets and boundaries are defined in [Security](SECURITY.md). “Mitigation” means intended risk reduction, not a guarantee.

| Threat | Attacker capability | Impact | Mitigation | Residual risk |
| --- | --- | --- | --- | --- |
| Malicious device on same Wi-Fi | Advertise, connect, send arbitrary bytes | Spam, exploit parsers, trick user | Treat LAN as hostile; bounds, state validation, rate limits, explicit approval | User may approve deceptive peer; implementation bugs |
| Passive packet observer | Capture multicast and unicast | Metadata/content/token disclosure | Minimize discovery; TLS 1.3 early; secure-session AEAD | Traffic size/timing and advertised service remain visible |
| Active MITM | Intercept/modify/terminate connections | Token theft, identity substitution, content change | Reviewed token/PAKE binding, transcript-bound identities/version/KEX, confirmation | Unresolved construction is current critical risk |
| Replayed transfer request | Replay captured valid request | Repeated prompts/resource use | Fresh instance/request IDs, deadlines, transcript/session binding, replay cache | Cache lost on restart; DoS still possible |
| Replayed token submission | Replay proof/token | Unauthorized pairing | Unique token/pairing IDs, one use, proof bound to transcript and ephemeral keys | Plain submission before reviewed binding is weak |
| Token brute force | Make online/offline guesses | Unauthorized pairing | CSPRNG token, 60 s, 5 paced attempts, global backoff, PAKE to prevent offline verification | Distributed online guesses/usability limits |
| Device impersonation | Claim device ID/name or stolen key | Wrong peer accepted | Verify key possession; bind key to pairing; show fingerprint/name safely | New devices have no prior trust; stolen key succeeds cryptographically |
| Malicious mDNS advertisement | Forge records/addresses/capabilities | Redirect, fingerprint, connection DoS | Discovery is hint only; verify after connect; deduplicate cautiously | Spam and metadata poisoning remain availability issues |
| Protocol downgrade | Strip versions/capabilities/algorithms | Weaker profile | Include offered and selected values in authenticated transcript; fail critical mismatch | Bugs or policy allowing obsolete suite |
| Oversized messages | Claim huge frame/count/length | Memory/disk exhaustion | Header maximum before allocation, checked arithmetic, count/queue limits | Many small valid messages can still consume work |
| Decompression bomb | Supply compressed tiny input/huge output if added | CPU/memory/disk exhaustion | No compression in 1.0; later post-auth negotiation and output/ratio/time caps | Codec vulnerabilities |
| Filename traversal | Send `../`, absolute, prefix, separators | Arbitrary overwrite | Treat name as component, local root, normalize/reject, create-new/no-follow | OS-specific alias/canonicalization bugs |
| Symbolic-link attack | Race local destination links | Write outside root/overwrite | Reject symlinks, directory-handle-relative no-follow operations, random temp, revalidate | Platform APIs differ; hostile local user can race |
| Disk exhaustion | Claim/send huge data or many files | System disruption | Preflight space/policy, declared total cap, enforce received bytes, reserve/quota, cleanup | Free space can change; sparse/accounting quirks |
| Memory exhaustion | Connections/frames/queues | Crash/swapping | Global/per-peer caps, streaming, bounded channels, early length checks | Coordinated distributed clients |
| CPU exhaustion | Expensive handshakes/hashes/parsing | Unavailability | Cheap checks first, handshake concurrency cap, timeouts/backoff | Hashing legitimate huge files is costly |
| Connection flooding | Many TCP/QUIC attempts | FD/task exhaustion | Listener backlog, connection semaphore, per-source/global rate limits, short pre-auth timeout | Source spoofing/large botnet/local peers |
| Request spam | Valid small requests | Prompt fatigue/social engineering | Pending limit, mute/block UI, cooldown, one prompt per authenticated context | Names/addresses mutable; user fatigue |
| Corrupted file data | Flip bytes/send wrong offsets/hash | Bad file | AEAD, offset/order validation, independent received SHA-256, no finalization on mismatch | Malicious sender can intentionally send harmful but hash-consistent content |
| Compromised trusted device | Future trusted key acts maliciously | Auto-accept abuse | Trusted mode deferred; require scope, revocation, key-change warning, optional consent | Trust inherently expands impact |
| Stolen local identity key | Read identity private key | Device impersonation | OS key storage, permissions, rotation/revocation design, no export/logging | Software storage cannot defeat account compromise |
| Local malware | Read process/files/keys/UI | Full confidentiality/integrity loss | Least privilege, keychain, zeroization, safe temp permissions | Outside Lanweave security boundary |
| User accepts wrong device | Social engineering/similar names | Sends files to attacker | Receiver-generated token, show device context/fingerprint, explicit summaries | Human ceremony can fail; token relay possible without channel binding |
| Discovery metadata leakage | Observe stable ID/name/platform/version | Tracking/targeting | Omit stable IDs/platform/app version by default; ephemeral instance/pseudonym | Presence and service type remain visible |
| Malicious metadata/UI escape | Control chars/bidi/huge text | Spoofed prompts/log injection | UTF-8 and length rules, escape controls, visibly quote names, local error text | Unicode confusables remain; UX must warn |
| File changes during send | Local race or malicious local process | Hash/content mismatch, unintended bytes | Snapshot/handle identity checks, hash actual stream, detect size/mtime change, fail | No atomic snapshot on all filesystems |
| Replay/nonce reset after reconnect | Force state rollback | AEAD compromise or duplicated effects | New session keys/ID each connect; monotonic per-session sequence; never resume nonce state blindly | Faulty persistence/custom crypto |
| Error oracle | Probe detailed failures/timing | Token/key/state information | Coarse remote codes, constant-time checks where applicable, uniform close | Network timing cannot be perfectly hidden |

## Outside the boundary

Lanweave can reasonably mitigate hostile network input, passive/active network attackers after a reviewed handshake, unintended filesystem paths, corrupted transfer bytes, and bounded resource abuse. It cannot ensure the safety of file contents, protect data on a compromised endpoint, prove a user's real-world identity, prevent physical observation of the token, guarantee availability on filtered/jammed networks, or revoke a stolen identity until a revocation/trust design exists.

## Review triggers

Update this model whenever adding trusted devices, resume, compression, directories, relays, mobile background service, clipboard data, parallel streams, or automatic receipt. Each crosses a current boundary.
