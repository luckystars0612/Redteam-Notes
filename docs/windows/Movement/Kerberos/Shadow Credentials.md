## Theory
The Kerberos authentication protocol works with tickets in order to grant access. An ST (Service Ticket) can be obtained by presenting a TGT (Ticket Granting Ticket). That prior TGT can only be obtained by validating a first step named "pre-authentication" (except if that requirement is explicitly removed for some accounts, making them vulnerable to ASREProast). The pre-authentication can be validated symmetrically (with a DES, RC4, AES128 or AES256 key) or asymmetrically (with certificates). The asymmetrical way of pre-authenticating is called PKINIT.
!!! note
    The client has a public-private key pair, and encrypts the pre-authentication data with their private key, and the KDC decrypts it with the client’s public key. The KDC also has a public-private key pair, allowing for the exchange of a session key.
Active Directory user and computer objects have an attribute called `msDS-KeyCredentialLink` where raw public keys can be set. When trying to pre-authenticate with PKINIT, the KDC will check that the authenticating user has knowledge of the matching private key, and a TGT will be sent if there is a match.
## Exploit
1. be in a domain that supports PKINIT and containing at least one Domain Controller running Windows Server 2016 or above.
2. be in a domain where the Domain Controller(s) has its own key pair (for the session key exchange) (e.g. happens when AD CS is enabled or when a certificate authority (CA) is in place).
3. have control over an account that can edit the target object's msDs-KeyCredentialLink attribute.
=== "UNIX-like"
    From UNIX-like systems, the msDs-KeyCredentialLink attribute of a user or computer target can be manipulated with the [pyWhisker](https://github.com/ShutdownRepo/pywhisker) tool.
    ```bash
    pywhisker.py -d "FQDN_DOMAIN" -u "USER" -p "PASSWORD" --target "TARGET_SAMNAME" --action "list"
    ```
    When the public key has been set in the msDs-KeyCredentialLink of the target, the certificate generated can be used with Pass-the-Certificate to obtain a TGT and further access.
=== "Windows"
    From Windows systems, the msDs-KeyCredentialLink attribute of a target user or computer can be manipulated with the [Whisker](https://github.com/eladshamir/Whisker) tool.
    ```bash
    Whisker.exe add /target:"TARGET_SAMNAME" /domain:"FQDN_DOMAIN" /dc:"DOMAIN_CONTROLLER" /path:"cert.pfx" /password:"pfx-password"
    ```
    When the public key has been set in the msDs-KeyCredentialLink of the target, the certificate generated can be used with Pass-the-Certificate to obtain a TGT and further access.
!!! note
    **Self edit the KCL attribute**

    User objects can't edit their own `msDS-KeyCredentialLink` attribute while computer objects can. This means the following scenario could work: trigger an NTLM authentication from DC01, relay it to DC02, make pywhisker edit DC01's attribute to create a Kerberos PKINIT pre-authentication backdoor on it, and have persistent access to DC01 with PKINIT and pass-the-cache.

    Computer objects can only edit their own msDS-KeyCredentialLink attribute if KeyCredential is not set already.