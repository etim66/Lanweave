# Sequence Diagrams

## Pairing And Authorization

```mermaid
sequenceDiagram
    actor U1 as User 1
    participant I as Initiator app
    participant R as Responder app
    actor U2 as User 2
    U1->>I: Select visible device
    I->>R: TLS, hello, pair_request
    R->>U2: Show pairing request
    alt Reject
        U2-->>R: Reject
        R-->>I: pair_response(rejected)
        R-xI: Close
    else Accept
        U2-->>R: Accept
        R-->>I: pair_response(accepted)
        R->>R: Generate one-time code
        R->>U2: Display code
        U2->>U1: Show or read code privately
        U1->>I: Enter code
        I->>R: Initiator pairing share
        R-->>I: Responder pairing share
        I->>R: Initiator pairing confirmation
        R-->>I: Responder pairing confirmation
        Note over I,R: Authorized encrypted session
    end
```

The code is shown locally by the responder and entered locally by the initiator. It is not sent as a normal wire field.

## Transfer Request And Success

```mermaid
sequenceDiagram
    actor A as Requesting user
    participant Q as Requester app
    participant P as Recipient app
    actor B as Receiving user
    A->>Q: Paste paths, review, select Send
    Q->>P: transfer_request(names and sizes)
    P->>B: Show complete metadata
    B-->>P: Accept
    P->>P: Validate and prepare destination
    P-->>Q: transfer_response(accepted)
    P-->>Q: ready
    loop Each file in order
        Q->>P: DATA
        Q->>P: file_end(index, sha256)
        P->>P: Verify and safely finalize
        P-->>Q: file_result(verified)
    end
    Note over Q,P: Transfer ends, session returns to idle
```

## Transfer Rejection

```mermaid
sequenceDiagram
    participant Q as Requester
    participant P as Recipient
    actor U as Receiving user
    Q->>P: transfer_request
    P->>U: Show file names, sizes, count, total
    U-->>P: Reject
    P-->>Q: transfer_response(user_rejected)
    Note over Q,P: No DATA, session remains authorized and idle
```

## Reverse Transfer

```mermaid
sequenceDiagram
    participant I as Pairing initiator
    participant R as Pairing responder
    Note over I,R: Earlier transfer I to R completed
    R->>I: transfer_request
    I-->>R: transfer_response(accepted)
    I-->>R: ready
    R->>I: DATA and file_end
    I-->>R: file_result(verified)
    Note over I,R: Pairing roles stayed fixed, transfer roles changed
```

## Idle And Manual Close

```mermaid
sequenceDiagram
    participant A as Participant A
    participant B as Participant B
    alt Manual close
        A->>B: session_close(user_closed)
        A-xB: Close transport and clear authorization
    else 600 seconds with no transfer request
        A->>B: session_close(idle_timeout)
        A-xB: Close transport and clear authorization
    end
    Note over A,B: A later session repeats pairing, no trusted devices
```
