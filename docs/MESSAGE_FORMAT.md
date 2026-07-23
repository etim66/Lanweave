# Message Format

This document is normative for experimental protocol version 1. The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** describe interoperability requirements. The selected pairing schema is experimental; implementation and cryptographic review remain release-blocking.

## JSON profile

Control frames contain JSON with this fixed profile:

- The body MUST be UTF-8 without a BOM and contain exactly one top-level object. Trailing non-whitespace data is invalid.
- Every object MUST reject duplicate fields and fields not defined for that message. Fields are at the top level; there is no `payload` envelope.
- Object field order and insignificant whitespace have no semantic meaning, except the exact two `hello` body byte strings are retained without reserialization for the cryptographic pairing binding.
- Numbers MUST be integers from `0` through `9007199254740991` (`2^53-1`). Negative numbers, floats, exponent notation, `NaN`, and infinities are invalid.
- `null` is not used. A field is either required or omitted.
- Strings MUST decode to valid Unicode scalar values. A filename is additionally limited to 255 UTF-8 bytes. `display_name` is limited to 128 UTF-8 bytes. A `reason` or `code` is limited to 64 ASCII bytes.
- A field name is limited to 32 ASCII bytes, an object to 32 fields, an array to 1,024 elements, and nesting to 8 levels.
- A JSON frame body has a generic hard allocation limit of 1 MiB (1,048,576 bytes). Each exact `hello` body MUST NOT exceed 4,000 bytes, a `pairing` body MUST NOT exceed 4,096 bytes, and a `transfer_request` body MUST NOT exceed 256 KiB (262,144 bytes). Each limit includes JSON syntax and whitespace.
- Pairing binary values use canonical RFC 4648 URL-safe base64 without `=` padding. Decoders MUST reject padding, non-URL-safe alphabets, whitespace, non-canonical encodings, and the wrong encoded or decoded length.
- A SHA-256 value MUST be exactly 64 lowercase hexadecimal characters.

Implementations MUST apply these rules independently of their JSON library's defaults and MUST use checked arithmetic when summing sizes. The generic 1 MiB bound is enforced from the frame header before body allocation. A message-specific limit applies only after bounded parsing identifies `type`; state alone does not identify an unparsed JSON body because terminal controls may be valid in that state. An oversized `hello` or `pairing` is `invalid_message`; a structurally valid oversized `transfer_request` receives a rejecting `transfer_response` with reason `invalid_manifest`.

## Messages

The `type` field is required and has one of the exact lowercase values below. Unless stated otherwise, every listed field is required.

### `hello`

```json
{"type":"hello","version":1,"display_name":"Workstation"}
```

- `version` MUST be the integer `1`. An otherwise valid `hello` with another non-negative integer version receives `unsupported_version`; a missing or wrongly typed version is `invalid_message`.
- `display_name` is optional, untrusted text for display only.

The sender sends the first `hello`; the receiver sends the second. No capabilities, framing versions, limits, certificates, pairing codes, or algorithms are offered or selected. Each exact body is at most 4,000 bytes. Each peer MUST retain both exact JSON frame body byte strings through pairing. Pairing uses those original bytes, not a normalized or reserialized representation. With the AAD's fixed 67-byte overhead, the two bodies produce at most 8,067 bytes, below the RFC 9382 AAD limit of 8,176 bytes.

### `pairing`

Share:

```json
{"type":"pairing","step":"share","data":"BGsX0fLhLEJH-Lzm5WOkQPJ3A32BLeszoPShOUXYmMKWT-NC4v4af5uO5-tKfA-eFivOM1drMV7Oy7ZAaDe_UfU"}
```

Confirmation:

```json
{"type":"pairing","step":"confirm","data":"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"}
```

These values illustrate canonical encoding and exact length only; they are not one valid exchange transcript.

Each `pairing` object contains exactly `type`, `step`, and `data`:

- `step` MUST be exactly `share` or `confirm`.
- A `share` decodes to exactly 65 bytes and its unpadded base64url text is exactly 87 ASCII characters. The decoded value MUST be a valid SEC1 uncompressed P-256 point with the `0x04` prefix and pass the RFC 9382 validation required for the expected peer role.
- A `confirm` decodes to exactly 32 bytes and its unpadded base64url text is exactly 43 ASCII characters. It is the expected RFC 9382 HMAC-SHA256 key-confirmation tag.

Exactly four pairing records occur after both `hello` messages and before `transfer_request`:

1. sender Party A `share`;
2. receiver Party B `share`;
3. sender Party A `confirm`;
4. receiver Party B `confirm`.

Strict connection state and message direction determine the role. There is no role, code, attempt number, remaining-attempt count, algorithm, identity, certificate, exporter, or profile field. The pairing code never appears on the wire. An extra, duplicate, reversed, or wrong-state record is invalid and terminal.

The receiver sends its confirmation only after the sender confirmation validates and the code is atomically consumed. The sender MUST verify the receiver confirmation. Pairing is successful for a peer only when its entire outgoing framed confirmation has been accepted by the TLS writer and successfully flushed, and the peer confirmation required by its role has verified. Commitment does not prove peer delivery. A correctly encoded and ordered sender or receiver confirmation that fails cryptographic verification uses only `authentication_failed` when safe, or a silent close. Only a failed sender-confirmation verification admitted against an active ceremony affects the receiver's failed-confirmation counter. Ceremony unavailability, expiry, exhaustion, consumption, or nonexistence may produce that result or a silent close earlier in pairing; phase, timing, and availability are not hidden, and no dummy PAKE is performed merely to hide them.

### `transfer_request`

```json
{"type":"transfer_request","files":[{"name":"report.pdf","size":1048576},{"name":"empty.txt","size":0}]}
```

- `files` MUST contain 1 through 1,024 entries.
- Each entry MUST contain exactly `name` and `size`.
- `name` MUST be a non-empty filename component, not a path. It MUST NOT be `.`, `..`, contain `/`, `\`, NUL, or a control character, or exceed 255 UTF-8 bytes. The receiver also applies destination-platform filename rules.
- `size` is the exact byte count and MUST be at most `2^53-1`. Zero is valid.

Array order is file order. The file count and total byte count are derived from `files`; they are not transmitted. The checked total MUST also be at most `2^53-1`. The manifest is immutable after this message.

The sender sends `transfer_request` only after mutual pairing succeeds. The PAKE does not bind these previously unseen bytes; the now-authenticated TLS connection protects them.

### `transfer_response`

Accepted:

```json
{"type":"transfer_response","accepted":true}
```

Rejected:

```json
{"type":"transfer_response","accepted":false,"reason":"destination_exists"}
```

- `accepted` MUST be a boolean.
- `reason` MUST be present only when `accepted` is `false`.
- The version-1 rejection reasons are `user_rejected`, `invalid_manifest`, `invalid_filename`, `name_conflict`, `destination_exists`, `insufficient_storage`, `resource_limit`, and `unavailable`.

The receiver sends this message only after separately validating and presenting the complete post-pairing manifest for approval. Before sending `accepted: true`, it MUST also prepare the destination and required resources. A preparation failure sends `accepted: false` with the applicable existing reason. Acceptance applies to the whole ordered manifest. Version 1 has no subset response and no accepted-but-not-ready application failure state.

### `ready`

```json
{"type":"ready"}
```

After successful destination and resource preparation, the receiver sends the accepting `transfer_response` followed immediately by `ready`. It means the receiver is ready for DATA for manifest index 0. It does not confirm pairing, modify the manifest, or authorize DATA for any other state. The sender MUST wait for `ready` before sending DATA. Failure to write either message is a transport failure, not an application state in which an accepted request awaits later preparation.

### `file_end`

```json
{"type":"file_end","index":1,"sha256":"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"}
```

- `index` is the zero-based manifest index and MUST be the current file.
- `sha256` is the sender's digest of exactly the bytes sent for that file.

### `file_result`

Verified:

```json
{"type":"file_result","index":1,"status":"verified"}
```

Failed:

```json
{"type":"file_result","index":1,"status":"failed","code":"hash_mismatch"}
```

- `index` MUST identify the current file.
- `status` MUST be `verified` or `failed`.
- `code` MUST be absent for `verified` and present for `failed`.
- The version-1 failure codes are `size_mismatch`, `hash_mismatch`, `write_failed`, `destination_exists`, and `insufficient_storage`.

A receiver MAY send a failed result as soon as the current file fails, including while the sender is still sending DATA. It MUST delete the current temporary file before committing that result.

### `cancel`

```json
{"type":"cancel","code":"user_cancelled"}
```

The version-1 cancellation codes are `user_cancelled`, `source_unavailable`, and `shutdown`. Sending or receiving `cancel` is terminal for the connection.

### `error`

```json
{"type":"error","code":"unsupported_version"}
```

The version-1 error codes are `unsupported_version`, `invalid_message`, `authentication_failed`, `timeout`, `resource_limit`, and `internal_error`. Sending or receiving `error` is terminal. A peer SHOULD send one safe error when possible, then close; it MUST NOT send an error in response to an error. Unsafe framing errors close without a JSON response.

Wrong-code, exporter-binding, `hello`-binding, and otherwise valid sender-confirmation verification failures share `authentication_failed` or silent close. A correctly encoded and ordered receiver confirmation that fails sender-side verification has the same terminal outcome but never changes the receiver's counter. An unavailable, expired, exhausted, consumed, or nonexistent ceremony may use the same behavior earlier in pairing. Ceremony existence, expiry, protocol phase, timing, and availability are not promised to be indistinguishable, and implementations MUST NOT perform a dummy PAKE merely to hide them. No error or other message reports remaining attempts. All reason and code sets are closed. An unknown `transfer_response`, `file_result`, or `cancel` value is `invalid_message`. An unknown `error` code closes the connection without a response.

There are no message, request, transfer, session, correlation, or file IDs. One connection carries at most one transfer and protocol state supplies context.

## Framing

TCP and TLS are streams, so every JSON control object and binary file chunk uses this 12-byte header:

```text
offset  size  field
0       4     magic: ASCII "LNWV"
4       1     frame version: 1
5       1     kind: 0 = JSON, 1 = DATA
6       2     header length: unsigned big-endian 12
8       4     body length: unsigned big-endian
12      N     body: exactly body length bytes
```

There are no flags. Other magic values, frame versions, kinds, or header lengths are invalid. A JSON body is 1 through 1,048,576 bytes under the generic hard allocation bound. After bounded parsing identifies `type`, the 4,000-byte `hello`, 4,096-byte `pairing`, and 262,144-byte `transfer_request` limits apply. A DATA body is 1 through 1,048,576 raw file bytes; a zero-byte file has no DATA frame.

A receiver MUST accumulate partial headers and validate the complete header. For JSON it MUST reject a declared body over the generic 1 MiB limit before allocation; for DATA it MUST reject a body over the 1 MiB DATA limit or the current file's remaining bytes before allocation or payload processing. It then reads exactly the bounded declared body and performs a bounded parse to identify the JSON message before enforcing its message-specific limit. It MUST handle multiple frames in one read and MUST bound incomplete-frame buffers, parsed JSON allocations, in-flight pairing states, and queued outbound DATA. A malformed header or a body over a generic hard bound is terminal and the receiver MUST NOT read or allocate the claimed body. Pairing or `hello` excess is `invalid_message`; a structurally valid `transfer_request` excess is a rejecting `transfer_response` with reason `invalid_manifest`.

DATA is valid only while receiving the current file after `ready`. It contains no JSON, index, offset, or length field beyond the frame header. Its bytes implicitly continue the current manifest entry, and its body length MUST NOT exceed that file's remaining declared size.

TLS 1.3 is the sole record-protection layer for both JSON and DATA. SPAKE2 output does not encrypt frames or add an application AEAD. One TCP/TLS connection carries at most one transfer; control and DATA frames share that one ordered stream.
