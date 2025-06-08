### Enum all deleted user object
=== "Powershell"
    ```bash
    Get-ADObject -Filter 'isDeleted -eq $true -and objectClass -eq "user"' -IncludeDeletedObjects -Properties objectSid, lastKnownParent, ObjectGUID | Select-Object Name, ObjectGUID, objectSid, lastKnownParent | Format-List
    ```
### Restore deleted user
=== "Windows"
    ```bash
    Restore-ADObject -Identity '938182c3-bf0b-410a-9aaa-45c8e1a02ebf'
    ```
### Change password for a user
=== "Windows"
    - Powershell
    ```bash
    Set-ADAccountPassword -Identity cert_admin -Reset -NewPassword (ConvertToSecureString -AsPlainText "Password123!" -Force)
    ```
    - net
    ```bash
    net user cert_admin Password123! /domain
    ```
### Enable user
=== "Windows"
    ```bash
    Enable-ADAccount -Identity cert_admin
    ```