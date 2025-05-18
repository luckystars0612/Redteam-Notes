# LDAP
Ldap can be used to obtain a lot of information on an AD domain, enum users, groups, shares,...
There are multiple tools can use to enum it

## Common tools

=== "ldeep"

    The [ldeep](https://github.com/franc-pentest/ldeep) tool can be used to enumerate essential information like delegations, gpo, groups, machines, pso, trusts, users, and so on

    ```bash
    # remotely dump information 
    ldeep ldap -u "$USER" -p "$PASSWORD" -d "$DOMAIN" -s ldap://"$DC_IP" all "ldeepdump/$DOMAIN"

    # parse saved information (in this case, enumerate trusts)
    ldeep cache -d "ldeepdump" -p "$DOMAIN" trusts
    ```

=== "ldapsearch"

    [ldapsearch](https://github.com/openldap/openldap/tree/master) (C) quickly get various information from a domain controller through its LDAP service..

    ```bash
    # list naming contexts
    ldapsearch -h "$DC_IP" -x -s base namingcontexts
    ldapsearch -H "ldap://$DC_IP" -x -s base namingcontexts

    # enumerate info in a base (e.g. naming context = DC=DOMAIN,DC=LOCAL)
    ldapsearch -h "$DC_IP" -x -b "DC=DOMAIN,DC=LOCAL"
    ldapsearch -H "ldap://$TARGET" -x -b "DC=DOMAIN,DC=LOCAL"
    ```

=== "ldapsearch-ad"

    The [ldapsearch-ad](https://github.com/yaap7/ldapsearch-ad) Python script can also be used to enumerate essential information like domain admins that have their password set to never expire, default password policies and the ones found in GPOs, trusts, kerberoastable accounts, and so on.

    ```bash
    ldapsearch-ad --type all --server $DOMAIN_CONTROLLER --domain $DOMAIN --username $USER --password $PASSWORD
    ```
    The FFL (Forest Functional Level), DFL (Domain Functional Level), DCFL (Domain Controller Functionality Level) and naming contexts can be listed with the following command.
    ```bash
    ldapsearch-ad --type info --server $DOMAIN_CONTROLLER --domain $DOMAIN --username $USER --password $PASSWORD
    ```
=== "ldapdomaindump"

    [ldapdomaindump](https://github.com/dirkjanm/ldapdomaindump) is an Active Directory info dumper via LDAP, outputting human-readable HTML.

    ```bash
    ldapdomaindump --user 'DOMAIN\USER' --password $PASSWORD --outdir ldapdomaindump $DOMAIN_CONTROLLER
    ```

=== "ntlmrelayx"

    With Impacket's [ntlmrelayx](https://github.com/fortra/impacket/blob/master/examples/ntlmrelayx.py) (Python), it is possible to gather lots of information regarding the domain users and groups, the computers, ADCS, etc. through a NTLM authentication relayed within an LDAP session.

    ```bash
    ntlmrelayx -t "ldap://domaincontroller" --dump-adcs --dump-laps --dump-gmsa
    ```

## Netexec

NetExec (a.k.a nxc) is a network service exploitation tool that helps automate assessing the security of large networks. It can help us enum a lot of things like:

- map information regarding AD-CS (Active Directory Certificate Services)
- show subnets listed in AD-SS (Active Directory Sites and Services)
- list users, groups, shares,...

For more information, go to [netexec wiki](https://www.netexec.wiki/)
```bash
# list PKIs/CAs
nxc ldap "domain_controller" -d "domain" -u "user" -p "password" -M adcs

# list subnets referenced in AD-SS
nxc ldap "domain_controller" -d "domain" -u "user" -p "password" -M subnets

# machine account quota
nxc ldap "domain_controller" -d "domain" -u "user" -p "password" -M maq

# users description
nxc ldap "domain_controller" -d "domain" -u "user" -p "password" -M get-desc-users
```

