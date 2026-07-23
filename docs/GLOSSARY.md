# Glossary

| Term | Meaning |
| --- | --- |
| TUI | Terminal user interface. The interactive screen opened by running `lanweave`. |
| Command list | The list shown when a user enters `/`. It contains actions available in the current app state. |
| Visible device | An untrusted discovery result for a device currently running Lanweave. |
| Pairing initiator | The participant who selects a device, requests pairing, and enters the code. |
| Pairing responder | The participant who accepts or rejects pairing and displays the locally generated code after acceptance. |
| Pairing code | A memory-only, one-time eight-digit code used to authorize one live connection. It is shown by the responder and entered by the initiator. It is never sent as a normal wire field. |
| Provisional TLS | The encrypted connection before code authorization. It encrypts traffic but does not yet prove that the intended peer is connected. |
| Authorized session | The live connection after pairing succeeds. It can carry several separately approved transfers in either direction. |
| Session idle | A period with no proposed or active transfer. The session closes after 600 consecutive idle seconds. |
| Transfer requester | The session participant who proposes and sends the files for one transfer. |
| Transfer recipient | The participant who reviews the request and saves accepted files. |
| Manifest | The ordered list of proposed filenames and exact sizes in one transfer request. |
| Transfer | One approved manifest and its file data. Finishing a transfer does not finish the session. |
| Transfer rejection | The recipient declines a proposed manifest. The session remains active. |
| `pair_request` | A control message asking the selected peer to begin pairing. |
| `pair_response` | A control message accepting or rejecting a pairing request. |
| `pairing` | A cryptographic share or confirmation record used after the responder accepts and the initiator enters the code. |
| `transfer_request` | A control message proposing one complete manifest. |
| `transfer_response` | A control message accepting or rejecting that manifest. |
| `ready` | The recipient is prepared to receive file data for an accepted transfer. |
| `DATA` | A binary frame carrying bytes for the current file. |
| `file_end` | Marks the end of one file and provides its SHA-256 digest. |
| `file_result` | Reports whether the current file was checked and finalized or failed. |
| Verified prefix | Files completed before a later file in the same transfer failed. These completed files remain saved. |
| `transfer_cancel` | Stops a proposal or transfer. Before `ready` the session can remain open; after `ready` it closes. |
| `session_close` | Ends the authorized session. A future connection must pair again. |
| Trusted device | A device allowed to skip pairing in a later session. Lanweave does not support trusted devices. |
