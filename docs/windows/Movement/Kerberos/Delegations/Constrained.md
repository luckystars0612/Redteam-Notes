## Theory
If a service account, configured with constrained delegation to another service, is compromised, an attacker can impersonate any user (e.g. domain admin, except users protected against delegation) in the environment to access another service the initial one can delegate to.
!!! note 
    If the "impersonated" account is "is sensitive and cannot be delegated" or a member of the "Protected Users" group, the delegation will fail.

    The native, RID 500, "Administrator" account doesn't benefit from that restriction, even if it's added to the Protected Users group.

    Constrained delegation can be configured with or without protocol transition. Abuse methodology differs for each scenario. The paths differ but the result is the same: a Service Ticket to authenticate on a target service on behalf of a user.

    Once the final Service Ticket is obtained, it can be used with Pass-the-Ticket to access the target service.
## Exploit
### With protocol transition
If a service is configured with constrained delegation with protocol transition, then it can obtain a service ticket on behalf of a user by combining S4U2self and S4U2proxy requests, as long as the user is not sensitive for delegation, or a member of the "Protected Users" group. The service ticket can then be used with pass-the-ticket. This process is similar to resource-based contrained delegation exploitation.
=== "UNIX-like"
    From UNIX-like systems, Impacket's getST (Python) script can be used for that purpose.
    ```bash
    getST -spn "cifs/target" -impersonate "Administrator" "$DOMAIN"/"$USER":"$PASSWORD"
    ```
    !!! note
        ```bash
        [-] Kerberos SessionError: KDC_ERR_BADOPTION(KDC cannot accommodate requested option)
        [-] Probably SPN is not allowed to delegate by user user1 or initial TGT not forwardable
        ```
        When attempting to exploit that technique, if the error above triggers, it means that either

        - the account was sensitive for delegation, or a member of the "Protected Users" group.
        - or the constrained delegations are configured without protocol transition
=== "Windows"
    From Windows machines, Rubeus (C#) can be used to conduct a full S4U2 attack (S4U2self + S4U2proxy).
    ```bash
    Rubeus.exe s4u /nowrap /msdsspn:"cifs/target" /impersonateuser:"administrator" /domain:"domain" /user:"user" /password:"password"
    ```
### Without protocol transition
If a service is configured with constrained delegation without protocol transition (i.e. set with "Kerberos only"), then S4U2self requests won't result in forwardable service tickets, hence failing at providing the requirement for S4U2proxy to work.

This means the service cannot, by itself, obtain a forwardable ticket for a user to itself (i.e. what S4U2Self is used for). A service ticket will be obtained, but it won't be forwardable. And S4U2Proxy usually needs an forwardable ST to work.

There are two known ways attackers can use to bypass this and obtain a forwardable ticket, on behalf of a user, to the requesting service (i.e. what S4U2Self would be used for):

- By operating an RBCD attack on the service.
- By forcing or waiting for a user to authenticate to the service while a "Kerberos listener" is running.
### RBCD approach
The service account (called `serviceA`) configured for KCD needs to be configured for RBCD (Resource-Based Constrained Delegations). The service's `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute needs to be appended with an account controlled by the attacker (e.g. `serviceB`). The attacker-controlled account must meet the necessary requirements for service ticket requests (i.e. have at least one SPN, or have its `sAMAccountName` end with $).
#### 1. Full S4U2 (self + proxy)
The attacker can then proceed to a full S4U attack (S4U2self + S4U2proxy, a standard RBCD attack or KCD with protocol transition) to obtain a forwardable ST from a user to one of serviceA's SPNs, using serviceB's credentials.
=== "UNIX-like"
    From UNIX-like systems, Impacket's getST (Python) script can be used for that purpose.
    ```bash
    getST -spn "cifs/serviceA" -impersonate "administrator" "domain/serviceB:password"
    ```
=== "Windows"
    From Windows machines, Rubeus (C#) can be used to conduct a full S4U2 attack (S4U2self + S4U2proxy)
    ```bash
    Rubeus.exe s4u /nowrap /msdsspn:"cifs/target" /impersonateuser:"administrator" /domain:"domain" /user:"user" /password:"password"
    ```
#### 2.Additional S4U2proxy
Once the ticket is obtained, it can be used in a S4U2proxy request, made by serviceA, on behalf of the impersonated user, to obtain access to one of the services serviceA can delegate to.
=== "UNIX-like"
    From UNIX-like systems, Impacket's getST (Python) script can be used for that purpose.
    ```bash
    getST -spn "cifs/target" -impersonate "administrator" -additional-ticket "administrator.ccache" "domain/serviceA:password"
    ```
=== "Windows"
    From Windows machines, Rubeus (C#) can be used to obtain a Service Ticket through an S4U2proxy request, supplying as "additional ticket" the Service Ticket obtained before.    
    ```bash
    Rubeus.exe s4u /nowrap /msdsspn:"cifs/target" /impersonateuser:"administrator" /tgs:"base64 | file.kirbi" /domain:"domain" /user:"user" /password:"password"
    ```
!!! note
    Computer accounts can edit their own "rbcd attribute" (i.e. `msDS-AllowedToActOnBehalfOfOtherIdentity`). If the account configured with KCD without protocol transition is a computer, controlling another account to operate the RBCD approach is not needed. In this case, serviceB = serviceA, the computer account can be configured for a "self-rbcd".

    Nota bene: around Aug./Sept. 2022, Microsoft seems to have patched the "self-rbcd" approach, but relying on another account for the RBCD will still work.