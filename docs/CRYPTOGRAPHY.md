# Cryptography

## Status and rule

This document records what I expect each standard primitive to do. It is not a handshake design and it is not a security proof.

I will use reviewed libraries and established protocol constructions rather than implementing primitives myself. The part that joins pairing, TLS, identity, and the transcript still needs specialist review before I can describe any release as security-stable.

## Primitive roles

| Primitive/concept | Proposed Lanweave role | Important constraints |
| --- | --- | --- |
| Ed25519 | Persistent device identity signatures | Verify canonical transcript bytes; key possession is not receiver consent |
| X25519 | Ephemeral ECDH contribution for forward-secret key establishment | Validate via reviewed protocol/library; erase ephemeral private values |
| HKDF-SHA256 | Extract/expand shared material into independently labeled keys | Unique context labels; include transcript/session identities; never use raw ECDH output |
| ChaCha20-Poly1305 | Default AEAD for protected Lanweave records if application AEAD is needed | Unique nonce per key; authenticate headers/context as associated data |
| AES-GCM | Negotiable AEAD alternative | Requires reliable hardware/software performance and equally strict nonce uniqueness |
| SHA-256 | File integrity and transcript/fingerprint component where profile specifies | Unkeyed file hash is not peer authentication |
| Secure RNG | Device/ephemeral keys, UUIDv4, tokens, nonces/salts | OS CSPRNG; failure is fatal; never predictable PRNG |
| Fingerprint | Human/log-safe compact digest of canonical identity public key | Define algorithm/encoding; stable fingerprints enable tracking |

## Concepts

**Long-term device identity:** an Ed25519 key pair persisted per installation. Its public key is associated with device ID. Reinstallation/key rotation changes identity unless a separately designed migration exists.

**Ephemeral keys and forward secrecy:** fresh X25519 contributions per pairing/session prevent later theft of the long-term signing key from decrypting captured old traffic, assuming ephemeral secrets were erased and the reviewed handshake provides this property.

**Authenticated encryption:** AEAD ensures confidentiality and detects modification for ciphertext plus associated context. Authentication failure is terminal; do not expose which field failed.

**Nonces:** use an analyzed construction, preferably a per-direction monotonic 64-bit sequence encoded into the algorithm's nonce with a session-derived fixed prefix. Exhaustion, wrap, rollback, or reuse closes the session. Send/receive directions use different keys.

**Key rotation:** derive new traffic keys with labeled HKDF epochs before nonce exhaustion and optionally after byte/time thresholds. Version 1.0 may close and reconnect instead of in-session rekeying; exact thresholds are Needs Research.

## ChaCha20-Poly1305 versus AES-GCM

| Factor | ChaCha20-Poly1305 | AES-GCM |
| --- | --- | --- |
| Software performance | Consistent and strong without AES hardware | Can be slower without acceleration |
| Hardware acceleration | Less universal dedicated acceleration | Excellent on modern AES-capable CPUs |
| Side-channel implementation | Good constant-time software ecosystem | Safe library/hardware use required |
| Nonce misuse | Catastrophic/revealing under reuse | Also catastrophic under reuse |
| Interoperability | Standard TLS/AEAD option | Extremely widespread standard option |

My proposed application-level default is ChaCha20-Poly1305 because its performance is predictable without AES hardware. AES-GCM can be negotiated later; the exact AES key size still needs to be settled.

If TLS already owns record protection, I should use the TLS 1.3 cipher suite instead of adding another encryption layer without a clear purpose. Whichever layer makes the choice, the selected algorithms must be covered by the authenticated transcript so an attacker cannot downgrade them.

## Pairing token

The receiver creates the token with the operating system's secure random generator and displays it using an alphabet people can type reliably. `PairingTokenChallenge` carries the token ID, lifetime, attempt limit, pairing-session ID, and any public proof parameters. It never carries the token itself.

The token is used once and kept only as long as the pairing attempt needs it. If the chosen construction supports verifier material, I should avoid retaining the plaintext value.

The first schema permits direct submission because it gives an experimental implementation an explicit message to work with. It also has real costs: the channel endpoint sees the token, guesses must be rate-limited, and a logging or crash-dump mistake can expose it. Without channel confidentiality, a network observer can see it too. Hashing a short token does not fix this; it simply gives an attacker something to test guesses against offline.

Research priority is a reviewed PAKE or challenge-response proof that:

- does not reveal the token,
- limits an active attacker to online attempts,
- binds both peers' transcript/key-exchange views,
- fits available audited Rust libraries and independent implementations.

Plain token submission may be used only in an explicitly experimental prototype over a confidential channel; it is not the preferred security-stable design.

## Required transcript binding

Conceptually, a canonical transcript must cover:

```text
protocol/framing versions and required capabilities
both device public identities and device IDs
both instance IDs and roles
request ID, transfer ID, pairing-session ID, token ID
receiver approval and exact challenge parameters
token verification/proof outcome (never the plaintext token)
both ephemeral key contributions / TLS exporter context
selected algorithms and transport profile
ordered hashes of all handshake messages
secure-session ID and key-confirmation direction
```

Both peers sign/authenticate the same domain-separated transcript hash and perform explicit key confirmation. `SessionEstablished` is accepted only when the expected transcript hash and peer role match. Negotiation is included to prevent downgrade. Encoding must be canonical and length-delimited; string concatenation is forbidden.

This list states properties, not a safe construction. Specialist review must choose how TLS exporters, signatures, PAKE output, X25519, and HKDF compose—or determine that a standard protocol should replace part of the proposed message set.

## Key derivation properties

A reviewed profile should derive separate secrets for client-to-server records, server-to-client records, confirmation, exporter/resumption (if ever supported), and nonce prefixes. HKDF labels include `Lanweave`, protocol version, algorithm suite, secure-session ID, direction, and transcript hash. File hashes remain unkeyed SHA-256 because they verify received content, while AEAD protects chunks in transit.

## Storage and lifecycle

- Prefer OS keychain/credential APIs where available; a permission-restricted configuration file is a documented fallback, not equivalent protection.
- Never silently copy a private identity key through discovery or configuration export.
- Define atomic creation, permission checks, backup expectations, rotation, and lost-key UX.
- Locking memory and zeroization reduce exposure but do not defeat local malware or all compiler/OS copies.
- Do not log tokens, private keys, ECDH shared secrets, traffic/session keys, raw PAKE material, or decrypted malicious payloads.
- Public fingerprints and IDs may still be sensitive metadata; redact by default.

## Security review gates

1. TLS/PAKE/token construction and MITM resistance.
2. Canonical transcript encoding and downgrade binding.
3. Identity provisioning, certificate/raw-key model, and key-change behavior.
4. Nonce/sequence/key-update design and crash/reconnect behavior.
5. Constant-time verifier comparisons and error indistinguishability.
6. Library maintenance, misuse resistance, platform support, and test vectors.

Successful token verification alone does not establish cryptographic identity. A device key alone does not prove the user approved this transfer. Both must be bound to the same key exchange and session.
