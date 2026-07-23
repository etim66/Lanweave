# Sequence Diagrams

These diagrams show the fixed MVP pairing profile and the existing transfer
model. Pairing always precedes the manifest decision. All named controls are
JSON; file content is raw DATA protected only by TLS.

## Code Generation and Sharing

```mermaid
sequenceDiagram
    actor U as Receiving user
    participant R as Receiver ceremony manager
    U->>R: Start pairing ceremony
    R->>R: Uniform CSPRNG 8 ASCII decimal digits<br/>failures = 0; monotonic deadline = now + 120 s
    R-->>U: Display code for private out-of-band sharing
    Note over R: Memory-only, leading zeroes retained,<br/>active before any connection
```

## Pairing Success

```mermaid
sequenceDiagram
    actor U as Sending user
    participant S as Sender / SPAKE2 Party A
    participant R as Receiver
    U->>S: Enter privately shared code
    S->>R: TCP then TLS 1.3 (`lanweave/1`)
    Note over S,R: Exact TLS 1.3 profile completed through Finished;<br/>fresh one-certificate P-256 identity and exporter
    S->>R: sender `hello`
    R-->>S: receiver `hello`
    Note over S,R: Exporter + exact hello JSON bodies (no frame headers)<br/>form SPAKE2 AAD, not transcript TT; each hello <= 4,000 bytes
    S->>R: `pairing` sender share
    R-->>S: `pairing` receiver share
    S->>R: `pairing` sender confirmation (`cA`), framed and flushed
    R->>R: Admit and serialize; recheck deadline;<br/>verify code, transcript, exporter, and hellos
    R->>R: Atomically set `consumed_sender_confirmed`;<br/>best-effort cancel/zeroize other pairing states
    R-->>S: Complete framed receiver confirmation (`cB`)<br/>accepted by TLS writer; flush succeeds
    R->>R: Receiver connection becomes paired<br/>(flush is not a delivery guarantee)
    S->>S: Receive and verify complete `cB`;<br/>sender connection becomes paired
    Note over S,R: Mutual pairing success exists only after sender verifies `cB`;<br/>no manifest or DATA was sent before this point
```

## Wrong-Code Retry on a New Connection

```mermaid
sequenceDiagram
    participant S as Sender
    participant R as Receiver ceremony manager
    S->>R: TLS connection 1
    S->>R: sender `hello`
    R-->>S: receiver `hello`
    S->>R: sender share
    R-->>S: receiver share
    S->>R: syntactically valid sender confirmation (wrong code)
    R->>R: Serialized verification fails; failures = 1
    alt Safe coarse reply
        R-->>S: `error(authentication_failed)`
    else Silent failure
        R-xS: Close
    end
    Note over R: Ceremony and original deadline remain active
    S->>R: New TCP/TLS connection, fresh cert/exporter and SPAKE2 state
    S->>R: sender `hello`
    R-->>S: receiver `hello`
    S->>R: sender share
    R-->>S: receiver share
    S->>R: sender confirmation (correct code)
    R->>R: Serialized verification succeeds;<br/>set `consumed_sender_confirmed`
    R-->>S: receiver confirmation; TLS-writer flush succeeds
    S->>S: Verify receiver confirmation
    Note over S,R: Mutual pairing succeeds on connection 2
```

## Expiry or Exhaustion

```mermaid
sequenceDiagram
    participant S as Sender
    participant R as Receiver ceremony manager
    alt Monotonic deadline reaches 120 seconds
        R->>R: Consume ceremony as expired
        S->>R: sender share
        R-->>S: `authentication_failed` or silent close
        Note over S,R: No receiver share and no dummy PAKE are required
    else Failed confirmations across reconnects
        loop Each new TLS connection, attempts 1 through 4
            S->>R: sender `hello`
            R-->>S: receiver `hello`
            S->>R: sender share
            R-->>S: receiver share
            S->>R: syntactically valid invalid sender confirmation
            R->>R: Admit, verify, and atomically increment shared failure count
            R-->>S: `authentication_failed` or silent close
        end
        S->>R: New connection: valid sender share
        R-->>S: receiver share
        S->>R: syntactically valid invalid sender confirmation (attempt 5)
        R->>R: Admit and verify; atomically set failures = 5<br/>and `consumed_exhausted`; cancel/zeroize other states
        R-->>S: `authentication_failed` or silent close
        S->>R: Later valid sender share
        R-->>S: `authentication_failed` or silent close (no dummy PAKE)
    end
    Note over S,R: Authentication failures use the same coarse code,<br/>but phase, timing, and ceremony availability are not hidden
```

## Post-Pairing Manifest Rejection

```mermaid
sequenceDiagram
    actor U as Receiving user
    participant S as Sender
    participant R as Receiver
    Note over S,R: Pairing succeeded on this TLS connection
    S->>R: `transfer_request` (complete ordered manifest)
    R->>R: Validate names, totals, storage, and destinations
    alt Semantic or policy rejection
        R-->>S: `transfer_response` (rejected, reason)
    else User decision
        R->>U: Prompt for this complete manifest
        alt User rejects
            U-->>R: Reject
            R-->>S: `transfer_response` (rejected, `user_rejected`)
        else User accepts but destination preparation fails
            U-->>R: Accept
            R->>R: Prepare destination for index 0 and race-safe plan; failure
            R-->>S: `transfer_response` (rejected, applicable reason)
        end
    end
    Note over S,R: Terminal; pairing did not approve files;<br/>no `ready` or DATA
    R-xS: Close
```

## Successful Multi-File Transfer

```mermaid
sequenceDiagram
    actor U as Receiving user
    participant S as Sender
    participant R as Receiver
    Note over S,R: Pairing succeeded on this TLS connection
    S->>R: `transfer_request` (complete ordered manifest)
    R->>R: Validate the whole manifest
    R->>U: Prompt for the whole manifest
    U-->>R: Accept
    R->>R: Prepare destination for index 0<br/>and the race-safe destination plan
    R-->>S: `transfer_response` (accepted exactly as a whole)
    R-->>S: `ready`
    Note over S,R: Preparation completed before acceptance;<br/>only `ready` authorizes DATA
    loop For each manifest index in order
        loop Until exactly its declared byte count
            S->>R: raw DATA for current index
        end
        S->>R: `file_end(index, sha256)`
        R->>R: Verify size and hash; finalize temp
        R-->>S: `file_result(index, verified)`
        Note over S: Only this result permits the next index
    end
    Note over S,R: Final verified result completes the transfer
    S-xR: Close
```

## Active File Failure

```mermaid
sequenceDiagram
    participant S as Sender
    participant R as Receiver
    Note over S,R: Paired, manifest accepted, `ready` sent;<br/>index 0 already verified and finalized
    S->>R: raw DATA for index 1
    alt Write, storage, or destination failure while DATA is active
        R->>R: Stop and delete index 1 temp
        R-->>S: `file_result(1, failed, code)` before `file_end`
    else Digest failure after exact bytes
        S->>R: `file_end(1, sha256)`
        R->>R: Hash mismatch; delete index 1 temp
        R-->>S: `file_result(1, failed, hash_mismatch)`
    else Connection loss or cancellation
        S-->>R: `cancel`, or transport closes
        R->>R: Delete index 1 temp
    end
    Note over S,R: Terminal; finalized index 0 remains;<br/>later indices are never attempted
```
