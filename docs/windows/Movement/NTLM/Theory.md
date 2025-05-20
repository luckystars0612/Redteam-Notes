The LM (LAN Manager) and NTLM (New Technology LM) authentication protocols are widely used in today's Microsoft environments (but mostly NTLM). It relies on a challenge-response scheme based on three messages to authenticate. In order to prove its identity, the authenticating client is asked to compute a response based on multiple variables including:

- a random challenge sent by the server in a `CHALLENGE_MESSAGE`
- a secret key that is the hash of the user's password

NTLM uses a 3-message challenge-response protocol between a client and a server (or domain controller).

| **Step** | **Message**               | **Description**                                                                                                                                                   |
| -------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1️⃣      | **NEGOTIATE\_MESSAGE**    | The client initiates the connection and says: “I support NTLM.”                                                                                                   |
| 2️⃣      | **CHALLENGE\_MESSAGE**    | The server responds with: “Here’s a random challenge (8-byte nonce). Prove you know the password.”                                                                |
| 3️⃣      | **AUTHENTICATE\_MESSAGE** | The client sends a **response** computed from:<br>• The server’s challenge<br>• The hash of the user's password<br>• Additional metadata (username, domain, etc.) |

!!! note
    The client does not send the password, or even the hash, directly. Instead, it:

    - Takes the server’s challenge
    - Encrypts it using a hash of the password (either NTLM hash or HMAC-MD5 depending on version)
    - Sends back the result as a response
    - The server (or DC) performs the same computation using the known user password hash and compares it.

    **NTLMv1 vs NTLMv2 Differences**

    | Feature            | NTLMv1                                | NTLMv2                                      |
    | ------------------ | ------------------------------------- | ------------------------------------------- |
    | Security Level     | Weak (deprecated)                     | Stronger (still not ideal)                  |
    | Challenge Response | 16-byte response using DES            | HMAC-MD5 using username, domain, timestamp  |
    | Vulnerable To      | Relay attacks, brute-force, downgrade | Relay (still), but harder to crack          |
    | Used With          | Legacy systems                        | Modern Windows (when Kerberos not possible) |

The following table details the secret key used by each authentication protocols and the cryptographic algorithm used to compute the response

| **Authentication Protocol** | **Algorithm (for the Protocol)** | **Secret Key** |
|-----------------------------|----------------------------------|----------------|
| LM                          | DES-ECB                          | LM hash        |
| NTLM                        | DES-ECB                          | NT hash        |
| NTLMv2                      | HMAC-MD5                         | NT hash        |

The following table details the hashing algorithm used by each hashing format in Windows that allows the system to transform the user's password in a non-reversible format.

| **Hashing Format** | **Algorithm (for the Hash)**    |
|--------------------|----------------------------------|
| LM hash            | Based on DES (Data Encryption Standard) |
| NT hash            | MD4                              |
