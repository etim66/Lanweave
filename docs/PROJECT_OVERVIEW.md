# Project Overview

## Definition

I use “Lanweave” for two closely related things:

1. The **Lanweave Protocol**, an open, implementation-independent specification for secure local peer-to-peer file transfer.
2. A future Rust CLI, `lanweave`, serving as the initial reference implementation.

The protocol is the part other implementations should be able to rely on. Rust crate boundaries, Tokio tasks, terminal prompts, and OS adapters belong to the reference implementation unless the [Lanweave Protocol Specification](PROTOCOL.md) says otherwise.

## Problem

People routinely move a file across the room by sending it through the internet. Cloud drives and ad-hoc web tools are convenient, but they can require an account, expose content or metadata to another party, or lock both devices into one vendor's software. Removable media works, but it is a poor default for a task the local network can already carry.

Lanweave is meant to make the direct path inspectable and interoperable. Publishing the protocol gives independent implementations something concrete to agree on and test. It does not make the system secure by declaration; the design, implementation, and threat boundaries still have to stand up to review.

## Intended users and use cases

- People moving documents or media between their own desktop computers.
- Colleagues exchanging selected files on a shared local network.
- Developers building interoperable Lanweave Protocol implementations.
- Administrators evaluating a local-only transfer path.

Primary initial use cases are an explicitly approved single-file transfer and a bounded multi-file transfer between two desktops. Discovery-free direct IP entry, automation, and unattended receipt are not required initially.

## Design assumptions

The current design rests on the following assumptions. If one of them changes, the affected decisions need to be revisited rather than patched around quietly.

- Peers share IP connectivity but the LAN is hostile; local network membership is not authentication.
- The sender and receiver can compare information out of band by reading and typing a short token.
- Each installation can protect a persistent identity key at least as well as normal user credentials; hardware-backed storage is not assumed.
- System clocks may be wrong. Durations use monotonic clocks locally; remote timestamps are informational.
- The receiver chooses the destination root and overwrite policy. Remote paths never choose arbitrary local locations.
- Initial desktop targets are Linux, Windows, and macOS; behavior and library support require research on all three.
- mDNS may be blocked, stale, or unavailable. Discovery failure does not imply transport failure.
- The proposed wire protocol is version 1.0 while this document set is specification revision 0.1. See [Versioning](VERSIONING.md).
- The exact reviewed pairing-to-key-exchange construction is unresolved and is a release gate, not an invitation to invent cryptography.

## Principles

- **Local first:** content travels directly on the LAN in the initial profile.
- **No mandatory server:** discovery and transfer work without a cloud or rendezvous service.
- **Secure by default:** confidentiality, authenticated encryption, consent, validation, and resource bounds are baseline behavior.
- **Explicit consent:** a receiver approves the requested summary before pairing and transfer.
- **Simple human flow:** discovery, approve, compare/type, transfer, verify.
- **Extensible protocol:** explicit versions, capabilities, ignorable optional fields, and critical-field handling.
- **Cross-platform interoperability:** wire behavior is independent of OS path conventions and UI.
- **Conservative cryptography:** use reviewed protocols and library primitives; do not implement primitives manually.
- **Separation of concerns:** discovery, transport, framing, cryptography, protocol state, transfer, storage, and UI have explicit boundaries.
- **Fail closed:** ambiguity, invalid state, failed authentication, and hash mismatch do not result in a completed file.
- **Bounded work:** untrusted lengths and counts are checked before allocation or disk writes.

## Goals

- Discover active instances through `_lanweave._tcp.local.`.
- Maintain a persistent device identity and unique per-process instance identity.
- Negotiate compatible protocol versions and capabilities.
- Exchange metadata-only request summaries before receiver consent.
- Authorize a pairing with a receiver-generated, one-time, expiring token.
- Establish a pairing-authenticated encrypted session bound to identities and transcript.
- Stream one or more regular files with progress, backpressure, cancellation, and SHA-256 verification.
- Return structured, minimally revealing errors.
- Specify enough detail for independent implementations and compatibility tests.

## Non-goals and future scope

The initial version does not promise trusted-device auto-acceptance, resume, directories or folder sync, clipboard data, Android, GUI applications, relays, transfers to groups, Wi-Fi Direct, or Bluetooth-assisted discovery. It also does not provide anonymity, protection from local malware, end-user identity proof beyond the pairing ceremony, or guaranteed operation across isolated VLANs.

I may explore those features later, but they should not weaken consent or compatibility. Trusted devices in particular need a real design for revocation, key changes, compromise, and policy before they can safely bypass any prompt.

## Constraints

- LANs may have multiple interfaces, IPv4/IPv6, changing addresses, client isolation, multicast filtering, and firewalls.
- Human-entered tokens have low entropy and require rate limits plus a reviewed guessing-resistant binding.
- File names and sizes are attacker-controlled metadata.
- Very large files must stream with constant bounded memory.
- TCP preserves bytes, not application message boundaries.
- Multi-file atomicity across a whole transfer is generally unavailable on ordinary filesystems.
- A file can change after selection; the sender must detect or define snapshot semantics.

## Platforms

The reference CLI will start on Linux, Windows, and macOS. Week 1 research still has to confirm how mDNS, firewalls, key storage, and file operations behave on each one. Android and desktop GUIs can follow later. The protocol itself is not tied to those three platforms.

## Success criteria

My first useful milestone is deliberately small: two CLI processes on a controlled LAN transfer one file after explicit approval, authenticate the pairing, encrypt the session, verify the received bytes, and fail without leaving an unsafe file behind.

Early transport experiments may use explicit addresses so discovery does not block the security work. The first public milestone should include mDNS discovery.

Related documents: [Architecture](ARCHITECTURE.md), [Security](SECURITY.md), [Threat Model](THREAT_MODEL.md), and [Roadmap](IMPLEMENTATION_ROADMAP.md).
