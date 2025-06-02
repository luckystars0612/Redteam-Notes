AD CS is Microsoft’s PKI implementation that provides everything from encrypting file systems, to digital signatures, to user authentication (a large focus of our research), and more. While AD CS is not installed by default for Active Directory environments, from our experience in enterprise environments it is widely deployed, and the security ramifications of misconfigured certificate service instances are enormous.

AD CS and identified multiple theft, escalation and persistence vectors:

- Credential theft (dubbed THEFT1 to THEFT5)
- Account persistence (dubbed PERSIST1 to PERSIST3)
- Domain escalation (dubbed ESC1 to ESC14)
- based on misconfigured certificate templates
- based on dangerous CA configuration
- related to access control vulnerabilities
- based on an NTLM relay vulnerability related to the web and RPC endpoints of AD CS
- Domain persistence (dubbed DPERSIST1 to DPERSIST3)
- by forging certificates with a stolen CA certificates
- by trusting rogue CA certificates
- by maliciously creating vulnerable access controls
## Exploit
### Terminology
- PKI (Public Key Infrastructure) — a system to manage certificates/public key encryption
- AD CS (Active Directory Certificate Services) — Microsoft’s PKI implementation
- CA (Certificate Authority) — PKI server that issues certificates
- Enterprise CA — CA integrated with AD (as opposed to a standalone CA), offers certificate templates
Certificate Template — a collection of settings and policies that defines the contents of a certificate issued by an enterprise CA
- CSR (Certificate Signing Request) — a message sent to a CA to request a signed certificate
EKU (Extended/Enhanced Key Usage) — one or more object identifiers (OIDs) that define how a certificate can be used
- Application Policy — this does the same thing as EKUs, but with a few more options. Specific to Windows environments
### Recon
#### Cert Publishers
An initial indicator is the "Cert Publishers" built-in group whose members usually are the servers where AD CS is installed (i.e. PKI/CA).

=== "UNIX-like"
    ```bash
    net rpc group members "Cert Publishers" -U "DOMAIN"/"User"%"Password" -S "DomainController"
    ```
=== "Windows"
    ```bash
    net group "Cert Publishers" /domain
    ```
#### pKIEnrollmentService objects
Alternatively, information like the PKI's CA and DNS names can be gathered through LDAP.

=== "netexec"
    ```bash
    netexec ldap 'domaincontroller' -d 'domain' -u 'user' -p 'password' -M adcs
    ```
=== "windapsearch"
    ```bash
    windapsearch -m custom --filter '(objectCategory=pKIEnrollmentService)' --base 'CN=Configuration,DC=domain,DC=local' --attrs dn,dnshostname --dc 'domaincontroller' -d 'domain.local' -u 'user' -p 'password'
    ````
### Attack paths
=== "UNIX-like"
    From UNIX-like systems, the Certipy (Python) tool can be used to operate multiple attacks and enumeration operations.
    ```bash
    # enumerate and save text, json and bloodhound (original) outputs
    certipy find -u 'user@domain.local' -p 'password' -dc-ip 'DC_IP' -old-bloodhound

    # quickly spot vulnerable elements
    certipy find -u 'user@domain.local' -p 'password' -dc-ip 'DC_IP' -vulnerable -stdout
    ```
=== "Windows"
    From Windows systems, the Certify (C#) tool can be used to operate multiple attacks and enumeration operations.
    ```bash
    Certify.exe cas
    ```
### Abuse
The different domain escalation scenarios are detailed in the following parts.

- ESC1 to ESC3, ESC9, ESC10, ESC13, ESC14 and ESC15: Certificate Templates
- ESC6 and ESC12: Certificate Authority
- ESC4, ESC5 & ESC7: Access Controls
- ESC8, ESC11: Unsigned Endpoints
- ESC16: Security Extension Disabled on CA ([certipy wiki](https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation))