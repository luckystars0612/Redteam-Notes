## Get all certificate template
=== "certutil"
    list all certificate templates available for `autoenrollment` or `manual enrollment` in domain
    ```bash
    certutil -template
    ```
=== "Powershell"
    ```bash
    Get-ADObject -Filter 'ObjectClass -eq "pKICertificateTemplate"' -SearchBase "CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=tombwatcher,DC=htb" -Properties displayName,flags,msPKI-Template-Schema-Version
    ```
=== "ldapsearch"
    ```bash
    ldapsearch -x -H ldap://10.10.11.72 \
    -D "alfred@tombwatcher.htb" -w basketball \
    -b "CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=tombwatcher,DC=htb" \
    "(objectClass=pKICertificateTemplate)" displayName
    ```
=== "certipy"
    ```bash
    certipy find -u 'alfred' -p 'basketball' -target 10.10.11.72 -stdout
    ```
## Enum all objects inside ADCS OU
=== "Cypher"
    ```bash
    MATCH (n)
    WHERE n.distinguishedname CONTAINS "OU=ADCS,DC=TOMBWATCHER,DC=HTB"
    RETURN n.name, n.distinguishedname
    ```
