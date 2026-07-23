# Architecture

## Initial Shape

The initial implementation is one Rust binary crate. Internal modules provide boundaries without freezing public crate APIs:

```text
main / CLI
    application orchestration and transfer state
        ceremony manager
        protocol values and validation
        small pairing-library adapter
        transfer and filesystem safety
    discovery adapter
    framed TCP/TLS adapter
        JSON codec and frame parser
```

Suggested module names are descriptive, not promises of public APIs: `cli`, `app`, `ceremony`, `protocol`, `framing`, `transport`, `pairing`, `transfer`, `storage`, and `discovery`. Preferred implementation families are `rustls` 0.23, `tokio-rustls` 0.26, `rcgen` 0.14, and `zeroize`.

Do not split crates prematurely. A module becomes a separate crate only when actual reuse, platform isolation, independent fuzzing, or dependency enforcement justifies the cost.

## Dependency Direction

Policy and domain rules stay inward; operating-system and library details stay at adapters.

```text
CLI ---------> application/state <--------- discovery adapter
                       |
                       +--> ceremony manager
                       +--> protocol/validation
                       +--> transfer policy <--------- filesystem adapter
                       +--> pairing adapter <--------- audited RFC 9382 library
                       +--> framed connection <------- TCP/TLS + JSON adapters
```

- The ceremony manager outlives individual connections. It owns only memory-resident code state and atomically checks the 120-second monotonic deadline, a budget of five failed sender-confirmation verifications across reconnects, cancellation, and one-use state. Bounded in-flight PAKE states plus independent connection, source, and CPU rate limits constrain work before verification admission.
- Successful sender-confirmation verification moves a ceremony to `sender-confirmed`, not mutually paired. The receiver becomes paired only after its receiver confirmation (`cB`) is committed and flushed to TLS; the sender becomes paired only after its sender confirmation (`cA`) is committed and it verifies `cB`. Concurrent sender confirmations race through one atomic transition; all losers fail closed.
- The application state machine owns post-pairing consent, the immutable manifest, active-file order, and terminal outcome. Pairing success cannot create manifest approval.
- Protocol validation does not depend on CLI, sockets, mDNS, or filesystem APIs.
- Discovery only emits untrusted candidate endpoints; it is not part of an active connection's state.
- Transport carries bounded frames and reports I/O events; it does not infer approval or file success.
- Storage maps the locally approved manifest into a receiver-selected directory, prepares it before wire acceptance, and never invents overwrite or rename policy.
- The pairing adapter exposes RFC 9382 message and confirmation operations without exposing or implementing group arithmetic. It uses exact identities `lanweave-v1-sender` and `lanweave-v1-receiver`; the exporter and exact sender/receiver `hello` JSON body bytes, excluding frame headers, enter the specified SPAKE2 AAD for confirmation-key derivation and do not enter RFC transcript `TT`.
- A prototype adapter may depend on candidate-only `pakery-spake2`, `pakery-core`, and `pakery-crypto` with its `p256` and `spake2` features. It must discard returned `Ke`/`session_key` after confirmation rather than expose either to transport or transfer code.
- Transport permits only TLS 1.3 and creates an ephemeral certificate/key for each connection. The ephemeral identity is not a persistent or independently trusted device identity. Demonstrating erasure of every transport private-key representation and library copy is a review gate, not an assumed property.

## Wire Layers

Control and data share one ordered frame stream but have distinct payload formats:

```text
control value -> strict JSON bytes ----+
                                      +-> frame header {kind, length} -> TLS records -> TCP
file bytes ----------------> DATA -----+
```

Inbound processing reverses the layers:

```text
TCP -> TLS -> bounded frame parser -> control: strict JSON -> semantic/state validation
                                  +-> DATA: raw bytes ---> active-file writer
```

The frame kind determines whether a payload is JSON control data or raw `DATA`. JSON is never used to carry file bytes. TLS is the sole record-protection layer; the live TLS exporter and exact `hello` JSON bodies are inputs to the specified SPAKE2 AAD and confirmation-key derivation before the manifest is sent, not additions to `TT`. Exact manifest approval is a later, separate state transition.

## Connection And Transfer Ownership

After mutual pairing, that connection owns exactly one transfer. One application task owns mutable transfer state, including the manifest index and current temporary file. Socket readers, the single frame writer, and file I/O workers communicate with that owner through bounded channels. Disconnecting before sender confirmation does not reset the ceremony deadline or failed-verification count; disconnecting after sender-confirmed does not make the code reusable, even if `cB` is lost and mutual pairing never completes.

Cancelling a ceremony cancels every active PAKE state attached to it. On any ceremony consumption or terminal transition, the manager and connection tasks perform best-effort zeroization of the code, password scalar, ephemeral scalars, confirmation keys, exporter copy, and any candidate-library `Ke`/`session_key`, including states waiting to enter serialized sender-confirmation verification. Cancellation and zeroization are idempotent; no dummy PAKE is started for unavailable ceremonies.

Bounded channels provide backpressure and memory limits. A single writer serializes complete frames so control and `DATA` bytes cannot interleave and preserves FIFO order through each `file_end`. A terminal control may bypass only after queued nonterminal frames are discarded. File I/O may use blocking workers where required, but the network task must not block on disk or terminal input.

No application ACK window is needed: TCP supplies reliable ordering, while `file_result` reports the semantic result after receiver verification. Cancellation is idempotent and causes the current partial to be removed. On any file failure, later manifest entries are never opened.

The connection sequence is TLS 1.3, `hello`, four `pairing` records, `transfer_request`, separate receiver review and destination preparation, accepting or rejecting `transfer_response`, then `ready` and `DATA` after acceptance. The records are sender A share, receiver B share, sender A confirmation, and receiver B confirmation. No manifest bytes are sent before mutual pairing confirmation, no accepting response is sent before preparation succeeds, and no file bytes are sent before exact manifest approval.

## Filesystem Boundary

Before an accepting `transfer_response`, storage planning validates every manifest entry against the target platform and selected destination and prepares the resources needed to receive manifest index 0. Unsafe names, duplicate names, platform-equivalent names, existing destination entries, or preparation failure reject the whole request.

Each file is written to a non-final temporary entry. Only exact length and digest verification permit safe finalization. Once finalized, that file joins the verified prefix. A later failure does not remove that prefix and must remove the current partial.
