# Two-Month Implementation Roadmap

This is the working plan for the first two months of implementation. It is deliberately ordered around risk: I want to learn whether discovery and pairing are sound before spending time on polished transfer features. The dates are targets, not permission to step around a security or correctness problem. If the schedule becomes tight, a reliable single-file transfer wins.

## Week 1 — Research and specification

The first week is for testing the assumptions in these documents. I need hands-on results for mDNS, transport, serialization, key storage, and the pairing design, followed by an updated decision log, a canonical schema draft, and a short brief for cryptographic review.

This work depends on access to Linux, Windows, and macOS, plus primary documentation and security expertise. The two findings most likely to change the plan are an unsuitable PAKE/TLS binding and mDNS behavior that cannot be hidden behind one reasonable abstraction.

Week 1 is done when the encryption boundary is unambiguous, the open security questions are named, and Week 2 has a defensible scope. If time runs short, trim the survey of comparable projects and the optional discovery TXT work first.

## Week 2 — Protocol foundation

Set up the Rust workspace and build the protocol foundation: types, identifier wrappers, the CBOR profile, framing, validation, version registries, and the first state reducer. The useful output is not just compiling types; it is a set of golden vectors and an incremental parser that stays inside its limits on hostile input.

This week depends on settling DD-003 and the frame header. Canonical encoding and unknown-field handling may turn out to be harder than expected, and it would be easy to publish APIs before they have earned their shape.

I can move on when arbitrary read boundaries and malformed frames produce deterministic results, oversized input is rejected before allocation, and 1.x/2.0 negotiation tests behave as specified. Do not spend the week splitting seven crates, optimizing throughput, or designing multi-file payloads.

## Week 3 — Discovery and identity

Implement mDNS advertisement, browsing, and resolution alongside persistent device identity and per-process instance IDs. By the end of the week, the `discover` and `identity` prototypes should exercise real platform adapters and a key-provider interface rather than bypassing them.

This depends on DD-001, DD-004, and DD-013, and on having real systems available for testing. Firewall prompts, multicast restrictions, and a weak file-based key fallback are the main things to watch.

Discovery is ready when two machines can appear, restart, expire, and reappear correctly; spoofed TXT data never becomes identity; and key storage passes the platform permission checks. Public-key fingerprints, platform/application TXT records, and polished direct-address entry can wait.

## Week 4 — Transport and request workflow

Build the TCP/TLS connection lifecycle, version negotiation, framed messaging, and the metadata-only request/accept/reject flow. The concrete deliverable is a two-process exchange with timeouts, rate bounds, and structured errors.

The Week 2 parser and the TLS identity research must be in place first. The largest design risk is still the boundary between provisional TLS and authenticated pairing; the largest implementation risk is leaving tasks or resources behind during cancellation.

This phase is complete when incompatible major versions are rejected before a request, no request can create file-data or path side effects, and every message has tests for the states in which it is invalid. QUIC, reconnection, resume, and parallel connections are not part of this week.

## Week 5 — Pairing and secure sessions

Implement the receiver's token flow, attempt and expiry rules, the reviewed key-establishment profile, transcript binding, and key confirmation. This week should leave behind adversarial tests, logs checked for secrets, and a written record of the security review—not only a happy-path demo.

Work must not start from an improvised answer to DD-005 or DD-010. It depends on a reviewed construction, secure randomness, and the key-storage work from Week 3. Direct token submission may prove unsuitable, which makes this the most likely point for the schedule to move.

I will consider this phase done only when MITM, replay, downgrade, and incorrect-token cases fail closed; neither peer can send file data before both confirmations; and secrets do not appear in logs. If the construction is not ready, I will keep the transport prototype non-sensitive rather than fill the gap with custom cryptography. Trusted devices, key-rotation UI, and broad algorithm agility can wait.

## Week 6 — Secure single-file transfer

Add the manifest/readiness exchange and the first complete file path: sequential chunks, a bounded receive window, checkpoint acknowledgements, streaming SHA-256, cancellation, progress, and temporary-file finalization.

This depends on the Week 5 secure session and on finding safe file APIs for each platform. Disk-full behavior, local filesystem races, rename semantics, and hashing throughput deserve more attention than raw benchmark numbers.

The milestone passes when zero-byte, larger-than-memory, interrupted, changed-source, collision, illegal-name, and hash-mismatch cases all leave bounded memory use and no unverified final file. Pre-hashing, a second verification read, multiple files, resume, and parallel chunks are the first things to defer.

## Week 7 — Multi-file and hardening

Extend the stable single-file path to sequential multiple files, decide how partial completion is reported, and spend the rest of the week on integration tests, fuzzing, rate limits, and failure recovery.

Nothing here should begin until the single-file transfer is dependable. Multi-file work multiplies states and makes it easy to imply whole-transfer atomicity that ordinary filesystems cannot provide.

The result should have deterministic file ordering and failure reports, a reusable fault corpus, cross-process tests, and no known crashing or unbounded-allocation fuzz case. If the single-file path regresses, remove multi-file support from the milestone rather than weakening the earlier guarantees.

## Week 8 — Release preparation

Use the final week to bring the specification back in line with the code, publish compatibility vectors, run benchmarks, set up CI, and prepare an experimental release. The release material should include a repeatable demo, known security limitations, a packaging plan, a checklist, and a licence decision.

This depends on a stable, reviewed core and testing on all three desktop platforms. Packaging and signing often take longer than expected, and any unresolved audit finding may block the release.

The release candidate is ready when fresh machines can complete the approved single-file flow, the docs describe the bytes and behavior actually shipped, and every release gate passes. Multi-file polish, every possible package manager, and performance tuning can wait. I will not make unqualified security claims about an experimental release.

## Milestone hierarchy

1. Parser/state safety without file I/O.
2. Rejected/accepted request flow without secrets or content.
3. Reviewed pairing-authenticated secure session.
4. Reliable, integrity-verified single file.
5. Multiple files and release packaging.

If a feature has not cleared its gate, it stays disabled. A smaller release whose single-file path is understandable and dependable is more useful than a broad release full of half-finished branches.
