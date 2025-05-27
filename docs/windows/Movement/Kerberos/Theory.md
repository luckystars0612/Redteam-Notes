A Kerberos realm is a logical group of networked computers that share a common authentication database. The authentication database is used to store the Kerberos tickets that are issued to users and services when they authenticate to the network.

In a Kerberos environment, each networked computer is a member of a realm. The realm is responsible for authenticating users and services and for issuing and managing Kerberos tickets.

A Kerberos realm can be implemented on any type of network, including networks that are not running Windows. In fact, the Kerberos protocol was developed as an open standard and is used by many different types of operating systems and networks.

On a Windows network, a Kerberos realm is typically equivalent to a domain. Each domain in a Windows network is a member of a realm, and the domain controller is responsible for authenticating users and services and for issuing and managing Kerberos tickets.

However, on a non-Windows network, a Kerberos realm can be implemented independently of any domain structure. In this case, the Kerberos server is responsible for authenticating users and services and for issuing and managing Kerberos tickets
## Tickets
Kerberos is an authentication protocol based on tickets. It basically works like this (simplified process):

1. Client asks the KDC (Key Distribution Center, usually is a domain controller) for a TGT (Ticket Granting Ticket). One of the requesting user's keys is used for pre-authentication. The TGT is provided by the Authentication Service (AS). The client request is called AS-REQ, the answer is called AS-REP.
2. Client uses the TGT to ask the KDC for a ST (Service Ticket). That ticket is provided by the Ticket Granting Service (TGS). The client request is called TGS-REQ, the answer is called TGS-REP.
3. Client uses the ST (Service Ticket) to access a service. The client request to the service is called AP-REQ, the service answer is called AP-REP.
4. Both tickets (TGT and ST) usually contain an encrypted PAC (Privilege Authentication Certificate), a set of information that the target service will read to decide if the authentication user can access the service or not (user ID, group memberships and so on).

A Service Ticket (ST) allows access to a specific service.
!!! note
    **cname formats**

    When requesting a service ticket, the client (`cname`) specifies the service it wants to obtain access to by supplying it's sname, which can be one of 9 types ([RFC 4120 section 6.2](https://www.rfc-editor.org/rfc/rfc4120#section-6.2)). Shortly put, the following formats are supported:

    - servicePrincipalName
    - userPrincipalName
    - sAMAccountName
    - sAMAccountName@DomainNetBIOSName
    - sAMAccountName@DomainFQDN
    - DomainNetBIOSName\sAMAccountName
    - DomainFQDN\sAMAccountName
    Note that if you use the `SRV01` string as a sAMAccountName, and the `SRV01` account does not exist, and the `SRV01$` account exists, this name will be treated as a principal name of the `SRV01$` account.

    On a good note, if the service name is specified by something else than an SPN (i.e. SAN, UPN), Kerberos will basically deliver service tickets if the requested service

    - has a trailing $ in the requested SAN (`sAMAccountName`)
    - or has at least one SPN (`servicePrincipalName`)

    If the service ticket is requested through a specific U2U (User-to-User) request, then neither of the conditions above will be required, the target user can be specified by its UPN (`userPrincipalName`).

The TGT is used to ask for STs. TGTs can be obtained when supplying a valid secret key. That key can be one of the following 

| **Key Name (etype)** | **Encryption Algorithm** | **Key Derivation Method**                                 |
| -------------------- | ------------------------ | --------------------------------------------------------- |
| **DES**              | Data Encryption Standard | Derived from user's password (weak algorithm, deprecated) |
| **RC4**              | RC4-HMAC (aka etype 23)  | Key **equals** NT hash (no salt used)                     |
| **AES128**           | AES-128 HMAC-SHA1        | Derived from user's password **with salt** (PBKDF2)       |
| **AES256**           | AES-256 HMAC-SHA1        | Derived from user's password **with salt** (PBKDF2)       |
!!! note
    **Salt used for key derivation**

    - ***Users***: uppercase FQDN + case sensitive username = DOMAIN.LOCALuser
    - ***Computers***: uppercase FQDN + host + lowercase FQDN hostname without the trailing $ = DOMAIN.LOCALhostcomputer.domain.local

An attacker knowing a user's NT hash could use it to ask the KDC for a TGT (if RC4 key is accepted). This is called Overpass-the-hash.

Users are not the only ones whose NT hashes can be used to abuse Kerberos.

- A TGT is encrypted with the krbtgt account NT hash. An attacker knowing the krbtgt's NT hash can forge TGTs impersonating a domain admin. He can then request STs as a domain admin for any service. The attacker would have access to everything. This forged TGT is called a Golden ticket.
- A ST is encrypted with the service account's NT hash. An attacker knowing a service account's NT hash can use it to forge a Service ticket and obtain access to that service. This forged Service ticket is called a Silver ticket.

`Overpass-the-hash`, `silver ticket` and `golden ticket` attacks are used by attackers to obtain illegitimate tickets that can then be used to access services using Kerberos without knowing any password. This is called Pass-the-ticket.