Kerberos delegations allow services to access other services on behalf of domain users
## Types of delegation
- Unconstrained delegations (KUD): a service can impersonate users on any other service.
- Constrained delegations (KCD): a service can impersonate users on a set of services
- Resource based constrained delegations (RBCD) : a set of services can impersonate users on a service
!!! note
    With constrained and unconstrained delegations, the delegation attributes are set on the impersonating service (requires SeEnableDelegationPrivilege in the domain) whereas with RBCD, these attributes are set on the target service account itself (requires lower privileges).
## Enum
=== "UNIX-like"
    From UNIX-like systems, Impacket's findDelegation (Python) script can be used to find unconstrained, constrained (with or without protocol transition) and rbcd.
    ```bash
    findDelegation.py "DOMAIN"/"USER":"PASSWORD"
    ```
    ```bash
    findDelegation.py -user "account" "DOMAIN"/"USER":"PASSWORD"
    ```
=== "Windows"
    From Windows systems, BloodHound can be used to identify unconstrained and constrained delegation.
    ```bash
    // Unconstrained Delegation
    MATCH (c {unconstraineddelegation:true}) return c

    // Constrained Delegation (with Protocol Transition)
    MATCH (c) WHERE NOT c.allowedtodelegate IS NULL AND c.trustedtoauth=true return c

    // Constrained Delegation (without Protocol Transition)
    MATCH (c) WHERE NOT c.allowedtodelegate IS NULL AND c.trustedtoauth=false return c

    // Resource-Based Constrained Delegation
    MATCH p=(u)-[:AllowedToAct]->(c) RETURN p
    ```
    The Powershell Active Directory module also has a cmdlet that can be used to find delegation for a specific account.
    ```bash
    Get-ADComputer "Account" -Properties TrustedForDelegation, TrustedToAuthForDelegation,msDS-AllowedToDelegateTo,PrincipalsAllowedToDelegateToAccount
    ```
    
    | **Property**                                                                 | **Delegation Type**                                | **Description**                                                                 |
    |------------------------------------------------------------------------------|----------------------------------------------------|---------------------------------------------------------------------------------|
    | `TrustedForDelegation`                                                      | Unconstrained Delegation                           | Allows a service to impersonate users to **any** service.                      |
    | `TrustedToAuthForDelegation`                                                | Constrained Delegation with Protocol Transition    | Allows a service to impersonate users to **specific services** with protocol transition. |
    | `AllowedToDelegateTo`                                                       | Constrained Delegation                             | Lists the **specific services** to which impersonation is allowed.            |
    | `PrincipalsAllowedToDelegateToAccount` (`msDS-AllowedToActOnBehalfOfOtherIdentity`) | Resource-Based Constrained Delegation (RBCD)       | Specifies **which accounts** are allowed to delegate **to this account**.     |
