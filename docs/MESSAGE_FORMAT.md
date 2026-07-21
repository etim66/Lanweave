# Message Format

## Status and design goals

This is a language-neutral, strongly typed logical schema for proposed protocol 1.0. Rust structs/enums may represent it later, but Rust/Serde behavior is not the wire specification. Exact numeric registry values and the cryptographic handshake payload remain draft.

## Serialization evaluation

| Format | Strengths | Trade-offs for Lanweave |
| --- | --- | --- |
| CBOR | Compact binary, standardized data model, deterministic encoding rules, flexible schema | Must restrict duplicate keys, nesting, indefinite lengths, numeric keys, and canonical form; library behavior must be verified |
| MessagePack | Compact, broad implementations, simple model | No single equally strong canonical profile; extension/schema conventions must be designed |
| Protocol Buffers | Mature IDL, efficient generated types, good field evolution | Tooling/generated code; signatures/transcripts and unknown-field preservation require care; less inspectable without schema |
| postcard | Very compact Serde-oriented format, attractive in Rust | Rust-centric ecosystem, schema evolution/cross-language interoperability less established for this goal |
| JSON | Human-readable and ubiquitous | Larger; number/string ambiguity, duplicate keys, canonical signing, binary chunks, and parser abuse require extra rules |

My current choice for the first implementation is CBOR with deterministic encoding based on RFC 8949. The Lanweave profile will use fixed numeric field keys and definite lengths, reject duplicate keys, bound nesting and collection sizes, and avoid floating-point values in protocol-critical fields. The examples remain JSON-like because they are easier to read.

This is still a proposed choice. I want cross-language interoperability tests before freezing it.

## Types and conventions

- `ID` is a 16-byte UUIDv4 on wire; text examples use canonical UUID syntax.
- `u16/u32/u64` are non-negative fixed semantic ranges regardless of codec representation; overflow rejects.
- `bytes<N>` is an exact-length byte string; `bytes<=N` is bounded.
- `text<=N` is valid UTF-8 bounded in encoded bytes; controls may be further forbidden.
- `duration_ms` is a relative duration, not a trusted remote deadline.
- Arrays have explicit maxima. Maps reject duplicate keys.
- Hashes and keys are binary on wire; examples abbreviate them.

UUIDv4 is recommended because it is random, clock-independent, and does not disclose creation time. UUIDv7 sorts well but embeds time and relies on clock behavior; reserve it for privacy-reviewed local databases. IDs provide correlation/collision resistance, not authentication or secrecy.

## Common envelope

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `protocol_version` | `{major:u16, minor:u16}` | Yes | Negotiated wire version; negotiation messages carry proposed context | Must match current negotiated version after negotiation |
| `message_type` | closed/registry enum | Yes | Payload discriminator | Known and enabled in state/capability; unknown critical type rejects |
| `message_id` | ID | Yes | Unique logical message ID | Non-nil; not previously accepted in session/replay window |
| `correlation_id` | ID | No | Message/request being answered | Required by response definition; must identify expected object |
| `sequence` | u64 | Conditional | Per-direction secure-session record sequence | Required after key establishment; exact expected value; no reuse/wrap |
| `secure_session_id` | ID | Conditional | Active secure session | Required for secure messages and must match state |
| `sent_at_ms` | signed/unsigned integer | No | Sender wall-clock diagnostic only | Bounded representation; never sole expiry/replay evidence |
| `payload_length` | u32 | Yes | Serialized payload bytes inside envelope | Must equal decoded payload span and type limit; frame remains authoritative boundary |
| `flags` | bit set | No | Critical/response/final/protection hints | Reserved bits zero; flags cannot weaken message's required protection |
| `payload` | typed map | Yes | Message-specific fields | Strict type/schema/state validation |

Timestamps are untrusted: peers may have wrong or malicious clocks. Local receipt time plus monotonic timers enforce expiry. Replay prevention uses fresh challenges/session keys, transcript binding, sequence windows, consumed-token state, and limited replay caches—not timestamps or UUIDs alone.

## Identifier lifecycle

| Identifier | Created by | Lifetime and purpose |
| --- | --- | --- |
| Device ID | Installation | Persistent public identity handle, associated with device public key; not authorization |
| Instance ID | Each process | Process lifetime; distinguishes restart/concurrent instance |
| Message ID | Message sender | One logical message; responses correlate to it; retransmission policy must not create a new semantic action accidentally |
| Request ID | Sender | Proposal through reject/expiry or accepted transfer association |
| Token ID | Receiver | One challenge; public identifier consumed with token |
| Pairing-session ID | Receiver | Acceptance/token/key-establishment context through failure or secure confirmation |
| Secure-session ID | Handshake result/assigned and confirmed by both | One established key/transcript context; never reused after reconnect |
| Transfer ID | Receiver on acceptance | Approved manifest/data/completion context |
| File ID | Sender | Unique within transfer (SHOULD be globally random); one metadata/data/hash stream |

Every namespace uses a distinct typed wrapper even though binary representation is the same. Implementations MUST NOT compare a file ID as a transfer ID accidentally.

## Protection classes

- **P0 Unauthenticated:** mDNS and the minimal connection preface. No secrets or side effects.
- **P1 Provisional channel:** preferably TLS-encrypted, but pairing identity not yet authenticated. Input remains hostile.
- **P2 Secure session:** encrypted/authenticated and sequenced under the pairing-bound session.

P1 does not satisfy requirements that say “secure session.” If a transport profile cannot provide P1 confidentiality, direct token submission is forbidden unless a reviewed proof replaces it.

## Message behavior matrix

This table defines purpose, direction, valid state, expected response/timeout, and security for every message. Field validation follows in the next section.

| Message | Purpose; sender → receiver | Valid state | Expected response / timeout behavior | Security considerations |
| --- | --- | --- | --- | --- |
| `DeviceHello` | Identify instance and advertised identity/capabilities; either → peer | Connected, before negotiation | Peer `DeviceHello` then negotiation; connection deadline | P1 preferred; claims untrusted until pairing; no private data |
| `DeviceGoodbye` | Advisory planned departure; either → peer | Any connected non-terminal state | `SessionClosed` if secure, otherwise close; no wait required | P1/P2 matching state; never treat as proof of peer disappearance |
| `ProtocolNegotiation` | Offer version/frame/capability/crypto profiles; initiator → responder (or symmetric profile) | Negotiating | Accepted/rejected within 10 s | P1; all offers transcript-bound; no downgrade fallback outside offer |
| `ProtocolNegotiationAccepted` | Select common parameters; responder → initiator and confirmation if defined | Negotiating | Peer validation/confirmation; 10 s | Selection subset of offers; bind exact canonical bytes |
| `ProtocolNegotiationRejected` | Report incompatibility; responder → initiator | Negotiating | Graceful close within 5 s | Coarse ranges safe; no transfer handling |
| `TransferRequest` | Propose bounded metadata-only transfer; sender → receiver | Negotiated/Idle | Accepted/rejected before 120 s | P1; rate-limit; no content/local paths |
| `TransferAccepted` | Record explicit approval and assign transfer/pairing IDs; receiver → sender | Waiting decision | Challenge follows promptly; request timer stops | P1; must reflect local user action, exact request |
| `TransferRejected` | Decline request; receiver → sender | Waiting decision | Request terminal; connection may remain | P1; coarse reason avoids policy leak |
| `PairingTokenChallenge` | Announce token ID/expiry/attempts/proof parameters; receiver → sender | Accepted/token generated | Submission before 60 s | P1; MUST NOT contain token or offline-verifiable low-entropy material |
| `PairingTokenSubmission` | Submit token or negotiated proof; sender → receiver | Waiting token submission | Accepted/rejected before remaining deadline | P1 confidentiality required for direct token; attempts counted before expensive work |
| `PairingTokenAccepted` | Confirm authorization step; receiver → sender | Verifying token | Begin key establishment immediately | P1; does not itself establish identity/session; bind result to transcript |
| `PairingTokenRejected` | Generic bad/expired/exhausted result; receiver → sender | Verifying token | Retry only if attempts/deadline remain; otherwise fail/close | P1; avoid oracle/detail/timing leakage |
| `KeyExchangeInit` | Start selected reviewed AKE profile; initiator → responder | Token accepted/establishing | Response within 10 s | P1/handshake protection; opaque profile data, transcript/identities bound |
| `KeyExchangeResponse` | Continue/complete reviewed AKE; responder → initiator | Establishing | Session confirmation within 10 s | Same; validation failure erases secrets/closes |
| `SessionEstablished` | Explicit key/transcript confirmation; each → peer | Key material ready | Both confirmations required before metadata; 10 s | P2 handshake keys; secure session/role/transcript exact match |
| `FileMetadata` | Declare exact file manifest entry; sender → receiver | Secure, receiving metadata | All entries then `TransferReady`; session idle timeout | P2; names/sizes/hash hostile and bounded |
| `TransferReady` | Confirm manifest/destination/resources/window; receiver → sender | Metadata validated | First chunk/file completion; 30 s inactivity | P2; no local paths; commits receive policy only for this manifest |
| `FileChunk` | Carry contiguous file bytes; sender → receiver | Transferring current file | Checkpoint ACK; inactivity/window timers | P2; length/offset/window checked before write |
| `ChunkAcknowledgement` | Report highest contiguous accepted offset/window; receiver → sender | Transferring | Sender advances window; no direct response | P2; never trust beyond bytes sent/file size; not completion |
| `FileComplete` | Declare sender end length/hash; sender → receiver | All bytes sent for file | Verification; next metadata/file or transfer completion | P2; receiver independently compares; mismatch fails |
| `TransferComplete` | Report/confirm all files verified; receiver then sender confirmation | Verifying/all finalized | Correlated confirmation then close | P2; cannot be inferred from EOF/ACK; contains results not paths |
| `TransferCancelled` | Cancel active request/pairing/transfer; either → peer | Any non-terminal relevant state | Stop work, cleanup, optional close | P1 before session/P2 after; idempotent; race has deterministic precedence |
| `SessionClosed` | Request/ack graceful secure close; either → peer | Secure non-terminal or completing | Correlated ACK; transport closes within 5 s | P2; erase keys after send/ack/deadline; no reuse |
| `Ping` | Liveness probe; either → peer | Negotiated idle or secure active | `Pong` within 10 s | Same protection as state; rate-limit, no timer amplification |
| `Pong` | Correlated liveness reply; peer → pinger | Awaiting pong | Clear probe timer | Same protection; echo only nonce, not arbitrary payload |
| `Error` | Coarse structured failure; either → peer | State where offending context is parseable | State-specific; fatal closes | Same protection as context; never expose secrets/internal details; no error loops |

## Payload field tables

Envelope fields above apply to every message and are not repeated.

### `DeviceHello`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `device_id` | ID | Yes | Claimed persistent device handle | Non-nil; later must match authenticated key association |
| `instance_id` | ID | Yes | Current process instance | Non-nil; not local instance |
| `device_public_key` | bytes<=64 | Yes | Encoded identity public key/profile | Valid negotiated encoding; fingerprint/ID consistency checked later |
| `display_name` | text<=128 | No | Untrusted UI label | Escape controls/bidi policy; never identity |
| `application_version` | text<=64 | No | Diagnostic build version | Untrusted; safe characters preferred |

### `DeviceGoodbye`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `instance_id` | ID | Yes | Departing process | Must match connected peer |
| `reason` | enum | No | `Shutdown`, `Restart`, `Unavailable` | Known non-critical value or ignore |

### `ProtocolNegotiation`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `version_ranges` | array<=8 of ranges | Yes | Supported major/minor intervals | Sorted, non-overlapping, sane u16 endpoints |
| `frame_versions` | array<=8 u8 | Yes | Supported frame formats | Unique; includes header version used for negotiation profile |
| `capabilities` | array<=128 capability | Yes | Optional and required features/parameters | Unique IDs; parameter bytes<=4096 total |
| `handshake_profiles` | array<=16 enum | Yes | Reviewed channel/AKE profiles supported | Unique; experimental disabled by default |
| `algorithm_suites` | array<=16 enum | Yes | Permitted identity/KEX/KDF/AEAD/hash suites | Unique, locally allowed |
| `max_control_frame` | u32 | Yes | Offered receive maximum | 4 KiB–1 MiB |
| `max_data_payload` | u32 | Yes | Offered chunk maximum | 64 KiB–1 MiB |

### `ProtocolNegotiationAccepted`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `selected_version` | version | Yes | Common protocol version | In both offered ranges and local policy |
| `frame_version` | u8 | Yes | Selected framing | In both offers |
| `capabilities` | array<=128 capability | Yes | Selected optional/required set | Subset satisfying all required capabilities |
| `handshake_profile` | enum | Yes | Selected reviewed profile | Offered and enabled |
| `algorithm_suite` | enum | Yes | Selected algorithms | Offered, profile-compatible, non-deprecated |
| `control_frame_limit` | u32 | Yes | Effective minimum limit | ≤ both maxima and protocol max |
| `data_payload_limit` | u32 | Yes | Effective minimum chunk limit | Within legal range and both offers |
| `offer_hash` | bytes<32> | Yes | Hash of canonical offers/order/roles | Exact locally computed value |

### `ProtocolNegotiationRejected`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `reason` | enum | Yes | Version/capability/profile incompatibility | Coarse known value |
| `supported_versions` | array<=8 ranges | Yes | Responder range for diagnostics | Same validation as offer |
| `missing_capabilities` | array<=32 IDs | No | Unmet critical capabilities | Bounded; no implementation details |

### `TransferRequest`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `request_id` | ID | Yes | Proposal identity | Fresh/non-nil |
| `file_count` | u32 | Yes | Number of proposed files | 1–1024 and equals summaries length |
| `total_size` | u64 | Yes | Sum of exact proposed lengths | Checked sum of summaries; local policy |
| `summaries` | array<=1024 summary | Yes | `{file_id, display_name, size}` only | Unique file IDs; names<=255; checked sum; total metadata<=256 KiB |
| `note` | text<=512 | No | Sender note | Escape controls; not authorization |

### `TransferAccepted`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `request_id` | ID | Yes | Approved request | Matches pending request/correlation |
| `transfer_id` | ID | Yes | Assigned transfer | Fresh/non-nil |
| `pairing_session_id` | ID | Yes | Assigned pairing context | Fresh/non-nil |
| `accepted_file_ids` | array<=1024 ID | Yes | Approved set | Initially exactly request set/order; unique |
| `request_expires_in_ms` | u32 | No | Remaining advisory duration | ≤ local pending deadline; local timer authoritative |

### `TransferRejected`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `request_id` | ID | Yes | Rejected request | Matches pending request |
| `reason` | enum | Yes | `UserRejected`, `Policy`, `Busy`, `Invalid`, `Expired` | Known/coarse |
| `retry_after_ms` | u32 | No | Advisory cooldown | Bounded ≤24 h; not promise |

### `PairingTokenChallenge`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `token_id` | ID | Yes | Public challenge handle | Fresh/non-nil |
| `pairing_session_id` | ID | Yes | Binding context | Matches accepted pairing |
| `expires_in_ms` | u32 | Yes | Relative token lifetime | 1,000–60,000 recommended/max profile |
| `attempts_allowed` | u8 | Yes | Total submission budget | 1–5 |
| `verification_method` | enum | Yes | `DirectSubmission` or reviewed proof profile | Negotiated and allowed; direct requires P1 confidentiality |
| `challenge_data` | bytes<=4096 | Conditional | Public PAKE/proof parameters | Profile-valid; MUST NOT contain token or enable avoidable offline guessing |
| `token_format_hint` | enum | No | Human input alphabet/length hint | From negotiated safe registry; does not reveal value |

### `PairingTokenSubmission`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `token_id` | ID | Yes | Target challenge | Active, unconsumed, unexpired |
| `pairing_session_id` | ID | Yes | Pairing context | Exact match |
| `method` | enum | Yes | Negotiated verification method | Exact challenge method |
| `token` | secret text<=64 | Conditional | Manually entered token for direct method | Alphabet/length; never log; P1 confidential channel required |
| `proof` | bytes<=4096 | Conditional | Reviewed method's next proof/message | Profile-valid; exactly one of token/proof as method requires |
| `attempt_number` | u8 | Yes | Sender view of attempt | Must equal expected next attempt; receiver budget authoritative |

### `PairingTokenAccepted`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `token_id` | ID | Yes | Consumed successful challenge | Matches active token |
| `pairing_session_id` | ID | Yes | Pairing context | Exact match |
| `verification_binding` | bytes<32> | Yes | Domain-separated hash/PAKE binding for transcript | Locally recomputed under selected reviewed profile; never token hash alone |

### `PairingTokenRejected`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `token_id` | ID | Yes | Challenge attempted | Matches or generic handling without oracle |
| `pairing_session_id` | ID | Yes | Pairing context | Exact match |
| `reason` | enum | Yes | `Invalid`, `Expired`, `AttemptsExhausted` | Coarse; do not reveal proof detail |
| `attempts_remaining` | u8 | No | Remaining online attempts | 0–4; omit if method analysis recommends less leakage |

### `KeyExchangeInit`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `pairing_session_id` | ID | Yes | Approved pairing being upgraded | Exact active pairing |
| `handshake_profile` | enum | Yes | Selected reviewed construction | Equals negotiated profile |
| `handshake_step` | u8 | Yes | Profile step number | Exactly expected |
| `handshake_data` | bytes<=16KiB | Yes | Opaque standard-protocol message/contribution | Parsed only by selected reviewed library/profile; bounded |
| `transcript_hash` | bytes<32> | Yes | Lanweave pre-handshake transcript checkpoint | Exact canonical local value |

### `KeyExchangeResponse`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `pairing_session_id` | ID | Yes | Pairing context | Exact match |
| `handshake_profile` | enum | Yes | Selected construction | Exact match |
| `handshake_step` | u8 | Yes | Response step | Exactly expected |
| `handshake_data` | bytes<=16KiB | Yes | Opaque reviewed-profile response | Library/profile validation; bounded |
| `transcript_hash` | bytes<32> | Yes | Transcript including init | Exact local value |

These two messages intentionally do not standardize raw signature/ephemeral fields until DD-005/DD-010 selects a reviewed profile. A profile may require additional round trips; if so, protocol 1.0 schema/state must be revised rather than overloading fields ambiguously.

### `SessionEstablished`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `secure_session_id` | ID | Yes | New secure context | Fresh; both peers derive/agree same value |
| `pairing_session_id` | ID | Yes | Source pairing | Exact active pairing |
| `transcript_hash` | bytes<32> | Yes | Final authenticated transcript | Exact locally computed value |
| `peer_role` | enum | Yes | Sender/receiver confirmation direction | Expected opposite role |
| `key_confirmation` | bytes<=64 | Yes | Profile-specific confirmation authenticator | Constant-time/profile verify; direction-separated |

### `FileMetadata`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `transfer_id` | ID | Yes | Approved transfer | Active transfer |
| `file_id` | ID | Yes | File identity | In accepted summaries; unique/not already declared |
| `ordinal` | u32 | Yes | Sequential order | 0..file_count-1, no duplicates |
| `display_name` | text<=255 | Yes | Untrusted component suggestion | Matches approved summary or permitted canonical update; safe mapping |
| `size` | u64 | Yes | Exact byte length | Matches request summary; policy/storage/check arithmetic |
| `sha256` | bytes<32> | No | Precomputed exact-byte digest | If present must match `FileComplete` |
| `name_was_lossy` | bool | No | Source name replacement indicator | Default false |
| `media_type` | text<=127 | No | Informational MIME-style hint | Syntax/bounds; never auto-execute |

### `TransferReady`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `transfer_id` | ID | Yes | Manifest accepted | Active transfer/all metadata validated |
| `manifest_hash` | bytes<32> | Yes | Canonical metadata set hash | Exact local value |
| `chunk_size` | u32 | Yes | Effective maximum payload | Negotiated 64 KiB–1 MiB |
| `receive_window_bytes` | u32 | Yes | Maximum unacknowledged payload | ≥chunk size; bounded local policy |
| `ack_checkpoint_bytes` | u32 | Yes | Cumulative ACK threshold | ≤window; proposed 4 MiB |
| `overwrite_policy` | enum | No | Coarse `Reject`/`Renamed`/`ApprovedReplace` | No destination/path disclosure |

### `FileChunk`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `transfer_id` | ID | Yes | Active transfer | Exact match |
| `file_id` | ID | Yes | Current file | Exact expected ordinal/file |
| `offset` | u64 | Yes | First byte position | Equals next expected contiguous offset |
| `data` | bytes<=1MiB | Yes | File bytes | Non-empty except no chunk for zero file; ≤negotiated size/window/remaining |
| `end_hint` | bool | No | Sender expects final chunk | Must imply offset+length=size; not completion proof |

### `ChunkAcknowledgement`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `transfer_id` | ID | Yes | Transfer | Active match |
| `file_id` | ID | Yes | File | Current/recent file allowed by state |
| `next_offset` | u64 | Yes | Highest contiguous accepted byte end | Monotonic, ≤sent and declared size |
| `receive_window_bytes` | u32 | Yes | Updated application window | Bounded; zero only temporarily |
| `durable` | bool | Yes | Whether bytes are durably flushed | Must be false in initial profile unless capability negotiated |

### `FileComplete`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `transfer_id` | ID | Yes | Transfer | Active match |
| `file_id` | ID | Yes | Completed file | Current file; all bytes sent/received before receiver accepts |
| `final_size` | u64 | Yes | Sender observed stream length | Equals metadata/request and final offset |
| `sha256` | bytes<32> | Yes | Sender hash of transmitted bytes | Equals metadata hash if present; receiver independently compares |

### `TransferComplete`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `transfer_id` | ID | Yes | Completed transfer | All accepted files verified/finalized |
| `manifest_hash` | bytes<32> | Yes | Completed manifest identity | Matches `TransferReady` context |
| `files_completed` | u32 | Yes | Count verified/finalized | Equals approved count |
| `total_bytes` | u64 | Yes | Total verified bytes | Equals checked manifest total |
| `confirmation` | bool | Yes | `false` receiver report; `true` sender acknowledgement | Direction/state exact; correlate acknowledgement |

### `TransferCancelled`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `scope` | enum | Yes | `Request`, `Pairing`, `Transfer` | Must match supplied ID/current state |
| `scope_id` | ID | Yes | ID being cancelled | Active and owned by context |
| `reason` | enum | Yes | `User`, `Timeout`, `Policy`, `Shutdown`, `PeerRequest` | Coarse known value |
| `last_file_id` | ID | No | Diagnostic current file | Must belong to transfer; no path |
| `last_offset` | u64 | No | Diagnostic contiguous progress | ≤declared size; not resume authorization |

### `SessionClosed`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `secure_session_id` | ID | Yes | Session closing | Active match |
| `reason` | enum | Yes | `Complete`, `Cancelled`, `Timeout`, `Shutdown`, `Error` | Coarse known value |
| `acknowledgement` | bool | Yes | False request / true response | Response correlates to close request; one round trip max |

### `Ping`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `probe_id` | ID | Yes | Liveness probe | Fresh; one bounded outstanding probe |
| `nonce` | bytes<16> | Yes | Random echo value | CSPRNG; not AEAD nonce/key material |

### `Pong`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `probe_id` | ID | Yes | Probe answered | Matches outstanding `Ping` and correlation |
| `nonce` | bytes<16> | Yes | Exact echoed probe bytes | Constant/exact comparison; no additional payload |

### `Error`

| Field | Type | Required | Description | Validation |
| ----- | ---- | -------: | ----------- | ---------- |
| `code` | error registry enum | Yes | Stable coarse code | Known; unknown handled as generic fatal semantics |
| `fatal` | bool | Yes | Sender will terminate affected scope | Receiver still applies local state rules |
| `scope` | enum | No | Connection/request/pairing/session/transfer/file | Consistent with scope ID/state |
| `scope_id` | ID | No | Affected object | Required when scope has ID and known |
| `retry_after_ms` | u32 | No | Advisory rate-limit delay | ≤24 h; local policy authoritative |
| `message` | text<=512 | No | Safe generic human hint | No secrets, paths, parser internals, or unescaped remote reflection |

## Logical examples

Examples omit the binary frame and abbreviate hashes. They are not JSON wire encodings.

```json
{
  "protocol_version": {"major": 1, "minor": 0},
  "message_type": "TransferRequest",
  "message_id": "7e36665e-4f70-4a93-8d7f-8f35fb21b561",
  "payload": {
    "request_id": "97836efe-7af8-41d6-9464-9b2cc7c18436",
    "file_count": 1,
    "total_size": 1048576,
    "summaries": [{"file_id": "abb8...", "display_name": "report.pdf", "size": 1048576}]
  }
}
```

```json
{
  "message_type": "PairingTokenChallenge",
  "correlation_id": "<TransferAccepted message ID>",
  "payload": {
    "token_id": "6fb0...",
    "pairing_session_id": "d32d...",
    "expires_in_ms": 60000,
    "attempts_allowed": 5,
    "verification_method": "DirectSubmission",
    "token_format_hint": "EightBase32Characters"
  }
}
```

Notice that the challenge has no token. Direct submission, if experimentally enabled, carries it only in the secret `token` field over a confidential provisional channel.

```json
{
  "message_type": "FileChunk",
  "secure_session_id": "31aa...",
  "sequence": 42,
  "payload": {"transfer_id": "b28f...", "file_id": "abb8...", "offset": 262144, "data": "<262144 bytes>"}
}
```

## Framing

Serialization does not delimit a TCP stream. Proposed version 1 frame header:

```text
offset  size  field
0       4     magic = "LNWV"
4       1     frame-format version = 1
5       1     flags (control/data, protected hint, critical; reserved bits zero)
6       2     header length = 12, unsigned big-endian
8       4     payload length, unsigned big-endian
12      N     exactly one serialized envelope
```

A four-byte big-endian length prefix would be smaller, but it gives the parser no magic value, framing version, or frame class, and makes an accidental cross-protocol connection harder to diagnose. I plan to use the 12-byte header for version 1. It has no CRC: a CRC is not authentication, and TLS or the session AEAD already detects protected corruption. If the header later needs to grow, that change gets a new frame-format version rather than a quiet reinterpretation of version 1.

Parser requirements:

- Accumulate fixed header across partial reads; never assume a read is one frame.
- Validate magic, version, header length, reserved flags, and class maximum before allocating payload.
- Control payload ≤1 MiB; data frame ≤negotiated chunk plus bounded envelope and never above proposed 1,048,640 bytes.
- Read exactly length bytes, parse exactly one envelope, reject trailing/unconsumed bytes and payload-length mismatch.
- Handle multiple complete frames already buffered without starving cancellation/control processing.
- On invalid/huge length, close without reading the claimed payload. Malformed CBOR, excessive nesting/collections, duplicate keys, or invalid type yields one bounded error only if safe.
- Limit buffered incomplete bytes and apply header/payload read deadlines to defeat slow sends.

Large file chunks remain bounded data frames; files are never one frame.

## Control and data mapping

Initial TCP profile uses one framed stream. A scheduler prioritizes control messages and limits queued chunk bytes; it cannot reorder wire bytes already written. Separate TCP data connections add authentication, lifecycle, and firewall complexity and are not recommended initially. A future QUIC profile should use one control stream and one stream per file, with the same logical messages or a separately versioned data-stream header. Multiple parallel streams and resume remain capability-negotiated future features.
