# Error Handling

This document groups the version 1 outcomes defined in [Message Format](MESSAGE_FORMAT.md). Wire responses carry stable codes only. Paths, OS errors, stack traces, parser offsets, hashes, key material, pairing codes and transcripts, failure counts, expiry state, and free-form diagnostics remain local.

## Authentication Failure

Pairing authentication failures use one coarse code: `error` with
`authentication_failed` when a reply is safe, or a silent close otherwise.
This rule does not hide the protocol phase, response timing, or whether a
ceremony is currently available. The receiver does not run a dummy PAKE.

- Wrong pairing code.
- The ceremony expired before sender confirmation was admitted.
- The shared five-attempt budget was exhausted, including the fifth failure.
- The sender confirmation did not bind the expected TLS exporter or exact hello
  JSON body bytes.
- Any other SPAKE2 transcript, HKDF, HMAC, or confirmation failure for which an
  authentication response is safe.
- A syntactically valid receiver confirmation (`cB`) failed sender-side
  cryptographic verification.

The wire does not report attempts remaining, expiry, lockout, or the failed
cryptographic check in an error field. In particular, ceremony expiry does not
use `timeout`. An unavailable, expired, exhausted, cancelled, or already
sender-confirmed ceremony may fail when the sender share arrives, before the
receiver sends a share. A valid wrong-code, exporter-binding, or confirmation
failure occurs later, when `cA` is verified. These phase and timing differences
explicitly reveal ceremony availability and are permitted; only the error code
and fields are coarse. Receiver-side UI and local logs may distinguish causes
without including secrets.

Only a failed, admitted verification of a syntactically valid sender-confirmation
record (`cA`) charges the ceremony's failure budget. The receiver serializes
admission and verification across connections and atomically rechecks the
monotonic deadline and ceremony state, verifies the confirmation and bindings,
then commits `consumed_sender_confirmed` or one failure. The fifth failure
atomically commits both failure count 5 and `consumed_exhausted` before any
reply. Expiry, unavailability, disconnect before commit, malformed frames,
JSON, pairing records, and shares do not charge the five attempts.

The failure counter is not a resource-control mechanism. Implementations
separately bound active and in-flight pairing states, expensive cryptographic
work, queues, concurrency, and per-source and global CPU rates. Every ceremony
consumption transition best-effort cancels and zeroizes all other active pairing
states for that ceremony; no such state may subsequently be admitted.

An authentication failure terminates only its connection. The active ceremony
and its original deadline survive reconnects after failures one through four.
The ceremony is consumed as `consumed_sender_confirmed` on valid `cA`, or on
expiry, local cancellation, or the fifth failed admitted confirmation. Ceremony
consumption is not itself mutual pairing success: the receiver becomes paired
only after its complete framed `cB` is accepted by the TLS writer and flush
succeeds, while the sender becomes paired only after receiving and verifying
`cB`. Writer acceptance plus a successful flush is a local commit boundary, not
a delivery guarantee.

## Manifest Rejection

After pairing succeeds, `transfer_response` rejects the complete manifest before
file data. The order is whole-manifest validation, user acceptance, destination
preparation, an accepting `transfer_response`, and then `ready`. If destination
preparation fails after user acceptance, the receiver sends a rejecting
`transfer_response` with the applicable reason; it never sends acceptance first.

| Reason | Meaning |
| --- | --- |
| `user_rejected` | The receiving user declined the request. |
| `invalid_manifest` | A structurally valid request fails bounded semantic count, size, or checked-total admission. Wrong JSON shapes or types use `invalid_message`. |
| `invalid_filename` | A name is not a safe destination filename component. |
| `name_conflict` | Requested names are duplicate or platform-equivalent. |
| `destination_exists` | A requested destination already exists. |
| `insufficient_storage` | Preflight storage policy cannot accept the complete manifest. |
| `resource_limit` | A bounded local request or concurrency policy rejected the request. |
| `unavailable` | The receiver cannot accept a transfer now. |

Rejection is terminal for the connection. It is not an `error`, and version 1 does not offer subset acceptance, rename, or overwrite. Pairing success is not manifest approval; the receiver validates, prompts for, and accepts or rejects the manifest separately.

## File Failure

`file_result` reports failure of the current manifest index:

| Code | Meaning |
| --- | --- |
| `size_mismatch` | The sender sent excess DATA or ended the file before its declared size. |
| `hash_mismatch` | The receiver's SHA-256 differs from `file_end`. |
| `write_failed` | Writing, closing, syncing under local policy, or finalizing the temporary file failed. |
| `destination_exists` | A destination appeared after preflight and no-replace finalization refused to overwrite it. |
| `insufficient_storage` | Storage became unavailable while receiving the current file. |

The receiver MAY send a failed result while the sender is still producing DATA,
before any `file_end` for that file. A failed result is terminal: the sender
stops DATA, the receiver deletes the current temporary file, the verified prefix
remains, and later files are not attempted.

## Cancellation

`cancel` represents intentional cancellation of this connection in any
nonterminal state, or a source-side condition that prevents an accepted
transfer from continuing:

| Code | Meaning |
| --- | --- |
| `user_cancelled` | A local user cancelled the connection. |
| `source_unavailable` | After an accepting `transfer_response`, the sender cannot read a source exactly as accepted. It is sender-only and invalid before acceptance. |
| `shutdown` | The local application is shutting down. |

`user_cancelled` and `shutdown` may be used in any nonterminal connection state.
Sending or receiving a valid `cancel` is terminal. A peer does not acknowledge a
valid cancellation. A malformed or wrong-state `cancel`, including
`source_unavailable` before acceptance or an unknown code, receives `error` with
`invalid_message` when safe. Cancelling the receiver-wide ceremony is a local
action that consumes it; it need not produce a wire response on unrelated
connections.

## Protocol Error

`error` reports a connection-level protocol or security failure:

| Code | Meaning |
| --- | --- |
| `unsupported_version` | An otherwise valid `hello` contains a non-negative integer version other than `1`. Missing or wrongly typed versions use `invalid_message`. |
| `invalid_message` | A bounded control violates its schema, limits, role, or state, including `transfer_request` before pairing or DATA before `ready` (including before pairing or manifest acceptance). |
| `authentication_failed` | Pairing authentication failed and a single coarse response is safe. |
| `timeout` | A non-ceremony protocol step or DATA progress deadline expired. |
| `resource_limit` | A connection-level bounded resource policy was exceeded after request admission. |
| `internal_error` | A local failure prevents safe continuation and no file-specific result applies. |

`error` has only `type` and `code`. It has no fatal flag, scope, identifier, attempt count, or remote diagnostic string.

All registries are closed. An unknown rejection, file-result, or cancellation value is `invalid_message`. An unknown `error` code closes the connection without a response, which prevents an error loop.

Response precedence is deterministic:

- Unsafe framing and hard frame-bound failures close without a response.
- Safely framed malformed JSON, wrong JSON types, unknown fields, malformed pairing shares or confirmations, invalid points or lengths, and wrong-state controls use `error` with `invalid_message` when safe and are subject to separate rate limits.
- `transfer_request` before pairing uses `error` with `invalid_message`; it is never treated as a manifest rejection.
- A safely parsed post-pairing `transfer_request` that fails semantic manifest admission uses rejecting `transfer_response`.
- A syntactically valid sender confirmation admitted against an active ceremony that fails cryptographic verification uses only `authentication_failed` or silent close. A correctly encoded and ordered receiver confirmation that fails sender-side verification uses the same terminal outcome but does not affect the receiver's failed-confirmation counter.
- An unavailable, expired, exhausted, cancelled, or sender-confirmed ceremony may use `authentication_failed` or silent close at sender-share processing, without generating a dummy receiver share.
- Excess DATA or premature `file_end` for the active current file uses failed `file_result` with `size_mismatch`.
- DATA outside an active receiving-file state uses `error` with `invalid_message`.

## Safe Reply Rules

An implementation sends `error` only when all of these hold:

- The TCP/TLS connection remains writable and framing boundaries are trusted.
- Enough bounded input was parsed to identify a violation without reflecting attacker-controlled detail.
- No terminal response has already been committed.
- The peer did not send a valid `error` or `cancel`.
- The code and fields do not encode the pairing, ceremony, or cryptographic
  cause; phase and timing may still reveal ceremony availability.

Malformed framing, impossible lengths, loss of frame synchronization, transport
close, and unsafe authentication failures close immediately without a JSON
response. Bounded malformed JSON with intact framing may receive `error` with
`invalid_message`. Implementations MUST NOT vary authentication error codes or
fields by authentication cause, but they need not conceal phase, timing, or
ceremony availability and MUST NOT fabricate dummy PAKE work for that purpose.

## Cleanup

A pre-transfer connection failure has no file cleanup and does not consume an
active ceremony unless `cA` committed `consumed_sender_confirmed`, or expiry,
cancellation, or the fifth failed admitted confirmation consumed it. On
ceremony consumption the receiver also best-effort cancels and zeroizes all
other pairing states for that ceremony. A failed `file_result`, `cancel`,
`error`, or active-transfer transport close prevents all later manifest
entries. The receiver closes and deletes the current temporary file and retains
finalized files in the verified prefix. The sender stops producing DATA
immediately. Cleanup is idempotent, and cleanup failures are logged locally
without exposing local paths to the peer.
