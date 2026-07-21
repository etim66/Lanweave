# Glossary

This glossary is normative for terminology in specification revision 0.1.

| Term | Definition |
| --- | --- |
| Advertisement | DNS-SD service instance and records published by a running Lanweave instance. It is a discovery hint, not proof of identity. |
| Authentication | Establishing that session messages are bound to the cryptographic keys and pairing context expected for the peer. It is distinct from authorization. |
| Authorization | Permission to perform an action; initially, the receiver's explicit approval plus successful one-time pairing. |
| Capability | A named optional protocol behavior that peers advertise and explicitly negotiate. |
| Chunk | A bounded, ordered byte range of one file identified by file ID, offset, and length. |
| Cipher | An encryption algorithm. Lanweave requires an authenticated-encryption construction, not encryption alone. |
| Control message | A small state/coordination message. Control frames have a lower size limit than data frames. |
| Data message | A message carrying file bytes, initially `FileChunk`. |
| Device ID | Stable, non-secret identifier for one installation, derived from or durably associated with its persistent device identity. It is not authorization by itself. |
| Device identity | Persistent Ed25519 key pair and its device ID association. |
| Discovery | Finding advertisements and resolving service instances to candidate addresses and ports. |
| File | One selected regular-file byte stream within a transfer, identified by a file ID. Directories and symbolic links are not files in version 1.0. |
| Frame | Length-delimited byte unit carrying one serialized Lanweave message over a stream transport. |
| Instance ID | Random identifier unique to one running process lifetime; it prevents confusing simultaneous/restarted instances. |
| Initiator | Peer that opens the transport connection, normally the sender. |
| Key agreement | Cryptographic process by which peers establish shared key material without transmitting the resulting session key. |
| Local device | Device on which the current implementation is running. |
| Message ID | Unique identifier for one logical message, used for diagnostics, duplicate detection, and correlation—not alone for replay prevention. |
| Nonce | Number used once with a particular AEAD key. Reuse under one key can be catastrophic. |
| Pairing | Human-authorized process binding receiver approval and token verification to the peers and key exchange. |
| Pairing session | Bounded pre-transfer context from token challenge through secure-session confirmation, identified by pairing-session ID. |
| Peer | Another Lanweave Protocol participant. Discovery does not make a device trustworthy. |
| Protocol | Rules, messages, states, validation, and security requirements that interoperable implementations follow. |
| Receiver | Peer asked to accept and write files. Normally the responder. |
| Request ID | Identifier for one transfer proposal, from `TransferRequest` until rejection, expiry, cancellation, or association with a transfer. |
| Responder | Peer accepting an incoming connection, normally the receiver. |
| Secure session | Negotiated, pairing-authenticated, confidential, integrity-protected protocol context with replay-resistant sequencing, identified by secure-session ID. |
| Sender | Peer that reads and transmits selected files. Normally the initiator. |
| Session ID | Context-specific identifier; documents use the precise term pairing-session ID or secure-session ID to avoid ambiguity. |
| Session identity | The secure-session ID plus authenticated peer identities, negotiated version/capabilities, transcript hash, and traffic-key context. |
| Token ID | Public random identifier for one token challenge. It is not the token and reveals no token value. |
| Transfer | One approved set of one or more files, identified by transfer ID. |
| Transfer ID | Unique identifier assigned after acceptance to the concrete transfer associated with a request. |
| Transfer request | Metadata-only proposal describing sender, file count, total bytes, and sanitized summaries; it contains no file content. |
| Transport | Ordered connection mechanism that carries frames; the proposed initial profile is TCP with TLS 1.3 channel protection. |
| Wire format | Exact bytes used for frames, serialized envelopes, messages, and cryptographic records. |

Additional identity terms are defined in [Message Format](MESSAGE_FORMAT.md); role and phase semantics are defined in [Protocol](PROTOCOL.md).
