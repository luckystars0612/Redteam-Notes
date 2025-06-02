Access privileges for resources in Active Directory Domain Services are usually granted through the use of an Access Control Entry (ACE). Access Control Entries describe the allowed and denied permissions for a principal (e.g. user, computer account) in Active Directory against a securable object (user, group, computer, container, organizational unit (OU), GPO and so on)

DACLs (Active Directory Discretionary Access Control Lists) are lists made of ACEs (Access Control Entries) that identify the users and groups that are allowed or denied access on an object. SACLs (Systems Access Control Lists) define the audit and monitoring rules over a securable object.

The following table should help for better understanding of the ACE types and what they allow.

| Common Name                     | Permission Value / GUID                         | Permission Type                        | Description |
|--------------------------------|--------------------------------------------------|----------------------------------------|-------------|
| **WriteDacl**                  | `ADS_RIGHT_WRITE_DAC`                           | Access Right                           | Edit the object's DACL (i.e. "inbound" permissions). |
| **GenericAll**                 | `ADS_RIGHT_GENERIC_ALL`                         | Access Right                           | Combination of almost all other rights. |
| **GenericWrite**               | `ADS_RIGHT_GENERIC_WRITE`                       | Access Right                           | Combination of write permissions (Self, WriteProperty), among other things. |
| **WriteProperty**             | `ADS_RIGHT_DS_WRITE_PROP`                       | Access Right                           | Edit one of the object's attributes. The attribute is referenced by an "ObjectType GUID". |
| **WriteOwner**                 | `ADS_RIGHT_WRITE_OWNER`                         | Access Right                           | Assume the ownership of the object (i.e. new owner of the victim = attacker). With "SeRestorePrivilege", it’s possible to specify an arbitrary owner. |
| **Self**                       | `ADS_RIGHT_DS_SELF`                             | Access Right                           | Perform "Validated writes" (i.e. edit an attribute's value and have it verified by AD). The attribute is referenced by an "ObjectType GUID". |
| **AllExtendedRights**          | `ADS_RIGHT_DS_CONTROL_ACCESS`                   | Access Right                           | Perform "Extended rights". This right can be restricted by specifying the extended right in the "ObjectType GUID". |
| **User-Force-Change-Password** | `00299570-246d-11d0-a768-00aa006e0529`          | Control Access Right (extended right)  | Change the password of the object without knowing the previous one. |
| **DS-Replication-Get-Changes** | `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2`          | Control Access Right (extended right)  | One of the two extended rights needed to operate a DCSync. |
| **DS-Replication-Get-Changes-All** | `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2`      | Control Access Right (extended right)  | One of the two extended rights needed to operate a DCSync. |
| **Self-Membership**            | `bf9679c0-0de6-11d0-a285-00aa003049e2`          | Validate Write                          | Edit the "member" attribute of the object. |
| **Validated-SPN**             | `f3a64788-5306-11d1-a9c5-0000f80367c1`          | Validate Write                          | Edit the "servicePrincipalName" attribute of the object. |
