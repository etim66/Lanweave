# State Machines

## MVP model

The receiver starts a pairing ceremony before accepting connections. Pairing
authenticates one TLS connection; only after pairing succeeds may the sender
send `transfer_request`. That request contains the complete ordered manifest of
regular files. The receiver then validates and prompts for that manifest as a
separate decision. Pairing never approves files.

One TCP/TLS connection carries at most one transfer. Control messages are JSON:
`hello`, `pairing`, `transfer_request`, `transfer_response`, `ready`, `file_end`,
`file_result`, `cancel`, and `error`. File bytes use raw DATA frames.

The fixed pairing profile is RFC 9382 SPAKE2 with P-256, SHA-256, HKDF, and HMAC.
The sender is Party A. It has exactly four pairing records in this order:
sender share, receiver share, sender confirmation, receiver confirmation. The
TLS exporter and the exact sender and receiver `hello` JSON body bytes, in wire
order and excluding their frame headers, are SPAKE2 additional authenticated
data (AAD). Each `hello` body is at most 4,000 bytes. The AAD is supplied to RFC
9382 confirmation-key derivation; it is not inserted into the RFC transcript
`TT`. Each connection has fresh SPAKE2 state and a fresh exporter. HKDF derives
the confirmation keys, and the HMAC confirmations authenticate the fixed
pairing transcript and AAD.

## Ceremony manager

The ceremony manager is receiver-wide and independent of per-connection state.
It owns the code, monotonic deadline, failure counter, and consumption result.

| State | Accepted event and action | Next state |
| --- | --- | --- |
| `inactive` | Local start: use a CSPRNG to sample uniformly from all 100,000,000 eight-character ASCII decimal strings; retain leading zeroes, set failures to 0, and set a monotonic deadline 120 seconds from generation | `active` |
| `active` | Local cancel; erase ceremony secrets | `consumed_cancelled` |
| `active` | Monotonic deadline reached; erase ceremony secrets | `consumed_expired` |
| `active` | Admit a syntactically valid sender confirmation (`cA`) for serialized verification; verification succeeds before the deadline, so atomically consume the ceremony, erase the code, and best-effort cancel and zeroize every other active pairing state for this ceremony | `consumed_sender_confirmed` |
| `active` | An admitted `cA` verification fails before the deadline and the atomically committed new failure count is 1 through 4 | `active` |
| `active` | An admitted `cA` verification fails before the deadline and the atomically committed new failure count is 5; erase ceremony secrets and best-effort cancel and zeroize every other active pairing state for this ceremony before any reply | `consumed_exhausted` |
| Any consumed state | No event reactivates or reuses the ceremony | Same state |

The code is memory-only: it is never persisted, logged, placed in discovery, or
sent by the receiver in a protocol message. The receiver displays it for private
out-of-band sharing. The 120-second deadline never pauses or extends.

All connections for an active ceremony share its five-failure budget. The
receiver serializes admission and sender-confirmation verification across
connections and, inside that serialization boundary, rechecks the deadline and
ceremony state, verifies `cA` and its exporter/hello binding, and atomically
commits either `consumed_sender_confirmed` or one failed verification. Only a
cryptographically failed, admitted `cA` verification increments the counter. A
connection loss before that verification commits, an unavailable ceremony, or
an expiry transition does not consume an attempt. Malformed shares,
confirmations, frames, or controls do not consume an attempt; they are governed
by separate bounded parsing and rate limits.

The five-attempt policy is not a resource bound. Implementations separately
bound concurrent and in-flight pairing states, queued confirmations, expensive
group/confirmation work, and per-source and global CPU rates. On any ceremony
consumption transition, cancellation and zeroization of other active pairing
states is best effort because work may already be running, but those states can
no longer be admitted against the consumed ceremony.

Wrong code, exporter/hello binding failure, or any other valid confirmation
failure has only `authentication_failed` or a silent close as its terminal
response. An expired, exhausted, cancelled, already sender-confirmed, or
otherwise unavailable ceremony may produce that same coarse outcome as soon as
the receiver processes the sender share; the receiver performs no dummy PAKE.
Consequently the failure phase and timing, and ceremony availability, are not
hidden. The receiver's local UI and logs may distinguish causes. An active
ceremony survives connection loss and failed attempts one through four, but
sender-confirmation consumption, expiry, local cancellation, or the fifth
failed admitted confirmation consumes it.

## Sender connection

| State | Accepted event and action | Next state |
| --- | --- | --- |
| `connecting` | The exact TLS 1.3 profile has completed through Finished with `lanweave/1`; send sender `hello` and retain its exact JSON body bytes | `awaiting_receiver_hello` |
| `awaiting_receiver_hello` | Valid compatible receiver `hello` of at most 4,000 body bytes; retain its exact JSON body bytes, obtain the TLS exporter, construct the fixed AAD, and send the Party A sender share | `awaiting_receiver_share` |
| `awaiting_receiver_share` | Valid receiver share in the fixed profile; derive the SPAKE2 result and send the sender confirmation | `awaiting_receiver_confirm` |
| `awaiting_receiver_confirm` | A complete valid receiver confirmation (`cB`) for the same SPAKE2 transcript, exporter, and exact hello bodies is received and verified after the sender's framed `cA` was committed and flushed | `paired` |
| `paired` | Send `transfer_request` with the complete manifest | `awaiting_transfer_response` |
| `awaiting_transfer_response` | `transfer_response` accepts the whole manifest | `awaiting_ready` |
| `awaiting_transfer_response` | `transfer_response` rejects it | `rejected` |
| `awaiting_ready` | `ready` | `sending_file` for manifest index 0 |
| `sending_file` | Send raw DATA for only the current index; after exactly its declared byte count, send `file_end(index, sha256)` | `awaiting_file_result` |
| `sending_file` | `file_result(index, failed, code)` from a receiver that cannot continue the current file | `failed` |
| `awaiting_file_result` | `file_result(index, verified)` and another manifest entry remains | `sending_file` for the next index |
| `awaiting_file_result` | Final `file_result(index, verified)` | `completed` |
| `awaiting_file_result` | `file_result(index, failed, code)` | `failed` |
| Any pairing state | `authentication_failed`, silent close, a valid confirmation authentication failure, timeout, or local cryptographic failure | `failed` |
| Any nonterminal state | Valid peer `cancel`, valid peer `error`, local connection cancellation, timeout, transport close, or local failure | Corresponding terminal state |

The sender never sends a manifest or DATA before `paired`, and never starts
index N+1 before receiving verified `file_result` for index N. Retrying pairing
uses a new connection, fresh TLS exporter, fresh receiver certificate, fresh
SPAKE2 ephemeral state, and the still-active out-of-band code.
Malformed receiver pairing input is `invalid_message` when a safe reply is
possible. A correctly encoded and ordered `cB` that fails cryptographic
verification is `authentication_failed`, not `invalid_message`, and it never
changes the receiver's failed-confirmation counter.

## Receiver connection

| State | Accepted event and action | Next state |
| --- | --- | --- |
| `awaiting_sender_hello` | Valid compatible sender `hello` of at most 4,000 body bytes; retain its exact JSON body bytes, send receiver `hello` of at most 4,000 body bytes, retain its exact JSON body bytes, and obtain the TLS exporter | `awaiting_sender_share` |
| `awaiting_sender_share` | Valid Party A sender share while the ceremony is active; construct the fixed AAD, derive the SPAKE2 result, and send the receiver share | `awaiting_sender_confirm` |
| `awaiting_sender_share` | Ceremony is expired, exhausted, cancelled, sender-confirmed, or otherwise unavailable; do no dummy SPAKE2 work, send only `authentication_failed` when safe, otherwise silently close | `failed` |
| `awaiting_sender_confirm` | Syntactically valid sender confirmation; ask the ceremony manager to perform serialized verification | `verifying_sender_confirm` |
| `verifying_sender_confirm` | Ceremony manager atomically enters `consumed_sender_confirmed`; queue the complete framed receiver confirmation only now | `committing_receiver_confirm` |
| `verifying_sender_confirm` | Ceremony manager reports failed authentication, expiry, or exhaustion; send only `authentication_failed` when safe, otherwise silently close | `failed` |
| `committing_receiver_confirm` | The complete framed receiver confirmation is accepted by the sole TLS writer in protocol order and that writer's flush succeeds | `paired_awaiting_request` |
| `committing_receiver_confirm` | The frame cannot be accepted or its flush fails, or the connection closes | `failed` with the ceremony already `consumed_sender_confirmed` |
| `paired_awaiting_request` | Valid `transfer_request`; validate the whole manifest and ask for one whole-manifest decision | `awaiting_decision` |
| `awaiting_decision` | Semantic or local rejection; send rejecting `transfer_response` | `rejected` |
| `awaiting_decision` | User accepts the exact manifest; begin destination preparation without sending acceptance | `preparing_destination` |
| `preparing_destination` | Destination preparation fails; send a rejecting `transfer_response` with the applicable reason | `rejected` |
| `preparing_destination` | Destination preparation for manifest index 0 and the race-safe destination plan succeeds; send accepting `transfer_response` | `accepted_awaiting_ready` |
| `accepted_awaiting_ready` | Send `ready` after the accepting response in writer order | `receiving_file` |
| `receiving_file` | Raw DATA while bytes remain for the current index; write to its temp file and hash it | `receiving_file` |
| `receiving_file` | Matching `file_end(index, sha256)` after exactly the declared bytes, including zero bytes | `verifying_file` |
| `receiving_file` | Current-file write, size, storage, or destination failure; delete the temp and send `file_result(index, failed, code)` when safe | `failed` |
| `verifying_file` | Final file succeeds; send `file_result(index, verified)` | `completed` |
| `verifying_file` | Current file succeeds and another entry remains; finalize it, advance the current index, and create the next temporary file before reporting progress | `preparing_next_file` |
| `preparing_next_file` | Next temporary file is ready; send the previous index's verified `file_result` | `receiving_file` for the new current index |
| `preparing_next_file` | Next temporary file creation fails; send the previous index's verified result followed by failed `file_result(new_index, code)` without accepting DATA | `failed` |
| `verifying_file` | Verification or finalization fails; delete the current temp and send `file_result(index, failed, code)` when safe | `failed` |
| Any nonterminal state | Valid peer `cancel`, valid peer `error`, local connection cancellation, timeout, transport close, or local failure | Corresponding terminal state |

A malformed sender share or confirmation is `invalid_message`, is rejected
before ceremony verification, and does not charge the failure budget. A valid
sender confirmation is the only event that permits the receiver confirmation.
An unavailable ceremony may instead fail at `awaiting_sender_share` with
`authentication_failed` or a silent close. A `transfer_request` in any state
before `paired_awaiting_request` is `invalid_message`; any manifest or DATA
before pairing, and any DATA before `ready`, is forbidden.

Before sending an accepting `transfer_response`, the receiver rejects the whole
manifest if an entry is not a valid filename component, if names are duplicates
or platform-equivalent, or if any destination already exists. User acceptance
therefore precedes destination preparation, and preparation success precedes
the accepting response. Preparation failure is a rejecting response, not an
accepted transfer followed by an error. No destination preparation remains to
be done between the accepting response and `ready`; nevertheless only `ready`
authorizes DATA. The sender is responsible for offering only regular files.
Destination creation must remain race-safe through finalization.

## Shared invariants

- Connection state, role, and the current manifest index correlate every input.
  There are no message, request, session, transfer, or file IDs.
- A control message or DATA frame is valid only in the state that lists it. The
  four pairing records cannot be skipped, repeated, reordered, or reflected.
- Valid `cA` consumes the ceremony as `consumed_sender_confirmed`; it is not by
  itself pairing success. The receiver connection becomes paired only after its
  complete framed `cB` is accepted by the TLS writer and flush succeeds. The
  sender becomes paired only after receiving and verifying `cB`. Pairing
  authenticates only this TLS connection; if it closes, it cannot be resumed or
  rebound.
- The manifest contains one or more regular files and is complete, ordered,
  immutable after `transfer_request`, and accepted exactly as a whole. No later
  metadata is allowed.
- Files are transferred sequentially. DATA belongs to the current manifest
  index and has no independent selector. There are no application chunk
  acknowledgements, windows, resume points, or concurrent files.
- A sender emits exactly the declared bytes, then `file_end(index, sha256)`, and
  waits for `file_result`. A verified result alone opens the next index. A failed
  result may arrive while DATA is still being sent and is terminal.
- A failed `file_result` is terminal. The receiver deletes the current temp,
  keeps the already verified prefix, and never attempts later files.
- The final verified `file_result` completes the transfer: on send for the
  receiver and on receipt for the sender. There is no transfer-complete or
  session-close message; the transport may then close.
- `cancel` means cancellation of this connection and is valid in any
  nonterminal connection state. `source_unavailable` is sender-only and valid
  only after an accepting `transfer_response`; other valid cancellation codes
  may terminate earlier phases. `cancel` and `error` are terminal. Transport
  close fails an active transfer. Cleanup removes only the current partial;
  verified files remain.
- There is no negotiation, ping/pong, goodbye, or second transfer on the same
  connection.

## Timeouts and races

- Each implementation uses bounded local deadlines for connection setup, peer
  decisions, pairing steps, readiness, DATA progress, `file_end`, and
  `file_result`. These do not extend the ceremony's fixed monotonic deadline.
- Ceremony expiry uses the same coarse `authentication_failed` outcome or a
  silent close, but its earlier phase and timing need not match a valid
  confirmation failure; it does not use the `timeout` code. Other protocol
  timeouts may send `error` with `timeout` only when a valid control reply
  remains safe.
- Serialize terminal events. The first committed terminal transition wins and
  cleanup is idempotent. Never answer `cancel` with `error` or `error` with
  `error`.
- For `cB`, committed means that the sole TLS writer accepted the complete JSON
  frame in protocol order and a flush operation returned success. This is a
  local transport boundary, not a delivery guarantee: the peer may never
  receive the bytes, and the sender reports pairing success only after it
  receives and verifies the complete `cB`.
- Local cancellation sends `cancel` if the control channel is writable, stops
  new DATA immediately, and cleans the current partial. A simultaneous peer
  cancellation has the same terminal result.
- Successful finalization is the retention boundary. Cancellation before
  finalization removes the temporary file; cancellation after finalization keeps
  the file even if its verified `file_result` was not delivered.
- If the receiver completes the final file but the final `file_result` is lost,
  the receiver remains completed while the sender fails on transport close or
  timeout. The MVP has no recovery handshake or resume.
