# Research Plan

Record the date, platform, versions, sources, reproducible steps, results, limits, licence, and affected design decision for every experiment.

## 1. TUI And Paste Behavior

- Compare `ratatui` with `crossterm` event handling on Linux, macOS, and Windows.
- Test bracketed paste, quoted paths, spaces, Unicode, multiple pasted paths, resize, suspend, panic cleanup, and terminal restoration.
- Check accessibility with keyboard-only navigation, clear focus, and screen-reader-friendly text output where possible.

## 2. Discovery Lifecycle

- Test `mdns-sd` registration, browse, goodbye, conflict, stale record, multi-interface, IPv4, scoped IPv6, and firewall behavior.
- Confirm that process exit removes the advertisement and that no helper daemon remains.
- Test direct-address fallback on multicast-blocked networks.

## 3. TLS And Pairing

- Confirm exact `rustls` APIs for TLS 1.3 only, ALPN, fresh certificates, disabled tickets/resumption/early data, exporter access, and custom certificate verification.
- Review the TLS exporter plus exact-hello binding against split-connection attacks.
- Evaluate only RFC 9382-conformant pairing libraries behind a small adapter. Check audit status, maintenance, constant-time behavior, vectors, dependencies, licence, and zeroization.
- Produce independent positive and negative vectors for initiator A and responder B.
- Test code generation after request acceptance, leading zeroes, 120-second expiry, one-attempt use, cancellation, and concurrency.

## 4. Session Protocol

- Prototype request accept/reject before code generation.
- Test several transfers over one connection and transfer direction changes.
- Test simultaneous proposals and verify both peers choose the initiator's request.
- Verify that pre-`ready` rejection returns to idle and that any failure after `ready` closes cleanly so in-flight DATA cannot enter a later transfer.
- Verify the 600-second monotonic idle rule with paused time.

## 5. Framing And Filesystem

- Fuzz strict JSON and incremental frame parsing with partial and combined reads.
- Test bounded writer queues and backpressure with files larger than memory.
- Test filename equivalence, no-follow creation, no-replace finalization, source mutation, disk full, and cleanup on supported platforms.

## Completion Criteria

- TUI and discovery behavior is demonstrated on all target desktop platforms.
- Session, collision, timeout, and reverse-transfer behavior has deterministic tests.
- Wire and filesystem prototypes have reproducible vectors and fault results.
- A reviewed pairing composition and independently audited dependency clear the release gate, or the project remains explicitly experimental and release-blocked.
