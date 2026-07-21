# Error Handling

## Model

Inside an implementation, errors should keep enough type and context to help the maintainer diagnose them. On the wire, they need to be much less revealing. When it is safe to reply, Lanweave maps a local failure to a bounded `Error` with a stable code, a fatal flag, optional retry advice, a correlation ID, and at most 512 bytes of locally written text.

An implementation never displays a peer's text without escaping it and never sends source chains, local paths, addresses, key details, parser offsets, or stack traces. Local errors are grouped into discovery, transport, protocol, validation, pairing, cryptographic, filesystem, transfer, cancellation, and internal categories.

A peer cannot force a state transition merely by setting `fatal=true`. The receiver first validates the error's context, then applies its own state-machine rules.

## Protocol-visible codes

Legend: **Remote** says whether a safe coarse code may be sent; **Recoverable** means recovery without starting a new request/session.

| Code | When | Remote | Recoverable | Transition / close | Local logging |
| --- | --- | :---: | :---: | --- | --- |
| `UnsupportedVersion` | No common major/minor or required capability | Yes | No | Negotiation → Failed; graceful close | Offered ranges/capabilities, peer endpoint; no secrets |
| `InvalidMessage` | Schema, canonical form, field, correlation, or bound invalid | Yes, once if parse/state safe | No | Current → Failed; close | Message type/ID, validation rule; bounded/redacted fields |
| `UnexpectedMessage` | Known message not allowed in current state/role | Yes | Usually no | Fail session; close on pre-auth/repeat | Expected/current state and type |
| `InvalidState` | Peer requests action inconsistent with session context | Yes, coarse | No | Fail affected operation; usually close | State/event and IDs |
| `InvalidRequest` | Request summary/count/size/policy invalid | Yes | New request possible | Request → Rejected/Failed; connection may remain | Rule and request ID |
| `ExpiredRequest` | Decision/submission after request deadline | Yes | New request | Pending request → Failed; connection may remain | Duration, request ID; not wall-clock claims |
| `RequestRejected` | Receiver user/policy rejects | Yes | New request | → Rejected; no forced close | Coarse reason only; local UI choice may be sensitive |
| `InvalidToken` | Token/proof fails with attempts left | Yes, indistinguishable | Retry within same challenge | Pairing → Waiting; do not close yet | Attempt number, token ID; never value |
| `ExpiredToken` | Challenge deadline elapsed | Yes | New approval/challenge | Pairing → Failed; consume token | Token ID and monotonic elapsed |
| `TooManyTokenAttempts` | Attempt budget exhausted | Yes | No in same request | Pairing → Failed; rate-limit and close | Counts/source context; no guesses |
| `AuthenticationFailed` | Signature, key confirmation, transcript, AEAD, identity fails | Optional generic only | No | Secure/session → Failed; erase secrets, immediate close | Stage and library error class; never key/material |
| `KeyExchangeFailed` | Negotiated handshake cannot complete without revealing auth detail | Optional generic | No | → Failed; erase/close | Profile, stage, public algorithm IDs |
| `SessionTimeout` | Idle/handshake deadline reached | Yes if channel valid | New session | → Failed; close | Timer/state/activity class |
| `DeviceBusy` | Concurrency/pending policy reached | Yes with bounded retry hint | Retry later | Reject request/connection gracefully | Counters and policy, not other peer data |
| `InsufficientStorage` | Preflight or write reports no capacity | Yes | Possibly new smaller request | Transfer → Failed; close/cleanup partials | Required/available sizes if local policy permits; local path redacted |
| `InvalidFilename` | Name cannot map safely | Yes | Metadata may be resent only if protocol later allows; 1.0 no | Transfer → Failed before chunks | File ID and rule; sanitized name only |
| `FileUnavailable` | Sender source missing/changed/unreadable | Yes | No for file in 1.0 | Transfer → Failed/cancelled | Local OS error/path protected |
| `HashMismatch` | Received length/hash differs | Yes | No in 1.0 | File/transfer → Failed; never finalize | File ID, expected/actual hashes only in protected local debug policy |
| `TransferCancelled` | User or policy cancels | Yes via dedicated message preferred | No | → Cancelled; cleanup; close gracefully | Initiator and coarse reason |
| `FrameTooLarge` | Header length exceeds class maximum | Usually no payload needed | No | Close immediately, optionally minimal error | Claimed length/endpoint |
| `RateLimited` | Request/connection/work budget exceeded | Yes, coarse retry hint | Retry later | Reject operation; may close | Bucket/category and duration |
| `InternalError` | Unexpected local invariant/resource failure | Yes, generic only | No | Fail safely; close/cleanup | Full local diagnostic without secrets |

## Category handling

- **Discovery:** registration/browse/resolve errors affect visibility, not active sessions. Retry with backoff and show actionable local context.
- **Transport:** connect refused/timeout may try another resolved address; mid-session EOF/reset fails transfer. Do not map all transport errors to peer fault.
- **Protocol/validation:** one safely parsed non-cryptographic violation may receive an error; ambiguity/repetition closes.
- **Pairing:** preserve indistinguishable failure messages and fixed attempt accounting.
- **Cryptographic:** fail closed, erase keys, do not retry in the same transcript, avoid oracles.
- **Filesystem:** keep local path/OS details local; remote gets the semantic code only.
- **User cancellation:** idempotent and expected, not logged as an internal failure.
- **Internal:** no panic should leave finalized unverified files; top-level cleanup guards are required.

## Error-message rules

`Error` MUST have a fresh message ID and correlate to the rejected message/request/session when known. An error MUST NOT be sent in response to an unparseable header, another error if that would loop, an AEAD authentication failure, or after close. Unknown error codes are handled as `InternalError` semantics without trusting remote fatal/retry advice. See [Message Format](MESSAGE_FORMAT.md).
