## Theory
Skeleton key is a persistence attack used to set a master password on one or multiple Domain Controllers. The master password can then be used to authenticate as any user in the domain while they can still authenticate with their original password. It makes detecting this attack a difficult task since it doesn't disturb day-to-day usage in the domain.

Skeleton key injects itself into the LSASS process of a Domain Controller to create the master password. It requires Domain Admin rights and SeDebugPrivilege on the target (which are given by default to domain admins).

!!! note
    Since this attack is conducted in memory by reserving a region of memory with VirtualAllocEx() and by patching functions, the master password doesn't remain active after a DC reboot.

This attack currently supports NTLM and Kerberos (RC4 only) authentications. Below are a few explanation of how it works depending on the authentication protocol it injects a master key for.

Skeleton Key can be injected with the misc::skeleton command in Mimikatz. It works in every 64-bits Windows Server version up to 2019 (included).

Mimikatz must be either launched as `NT-AUTHORITY\SYSTEM` or be executed with a domain admin account on the Domain Controller. For the latter, debug privileges (`SeDebugPrivilege`) must be set for Mimikatz to work. This can be done with the `privilege::debug` command.
```bash
mimikatz "privilege::debug" "misc::skeleton"
```
!!! info
    By default, the master password injected is `mimikatz`.