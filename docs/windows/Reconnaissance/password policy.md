The easiest way to compromise accounts is to operate some password spraying. In order to fine-tune this, the password policy can be obtained. This policy can sometimes be enumerated with a null-session (i.e. an MS-RPC null session or an LDAP anonymous bind).
=== "UNIX-like"
    On UNIX-like systems, there are many alternatives that allow obtaining the password policy like [polenum](https://github.com/Wh1t3Fox/polenum) (Python), [NetExec](https://github.com/Pennyw0rth/NetExec) (Python), [ldapsearch-ad](https://github.com/yaap7/ldapsearch-ad) (Python) and [enum4linux](https://github.com/cddmp/enum4linux-ng)
    ```bash
    # polenum (obtained through MS-RPC)
    polenum -d $DOMAIN -u $USER -p $PASSWORD -d $DOMAIN

    # netexec (obtained through MS-RPC)
    nxc smb $DOMAIN_CONTROLLER -d $DOMAIN -u $USER -p $PASSWORD --pass-pol

    # ldapsearch-ad (obtained through LDAP)
    ldapsearch-ad.py -l $LDAP_SERVER -d $DOMAIN -u $USER -p $PASSWORD -t pass-pol

    # enum4linux-ng (obtained through MS-RPC)
    enum4linux-ng -P -w -u $USER -p $PASSWORD $DOMAIN_CONTROLLER
    ```
=== "Windows"
    From a domain-joined machine, the net cmdlet can be used to obtain the password policy.
    ```bash
    net accounts
    net accounts /domain
    ```
    From non-domain-joined machines, it can be done with [PowerView](https://powersploit.readthedocs.io/en/latest/Recon/) (Powershell).
    ```bash
    Get-DomainPolicy
    ```

    !!! note "Note"

        Accounts that lockout can be attacked with [sprayhound (credential spraying)](https://github.com/Hackndo/sprayhound) while those that don't can be directly bruteforced with [kerbrute (Kerberos pre-auth bruteforcing)](https://github.com/ropnop/kerbrute)

