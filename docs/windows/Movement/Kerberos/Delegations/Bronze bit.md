## Theory
When abusing Kerberos delegations, S4U extensions usually come into play. One of those extensions is S4U2proxy. Constrained and Resource-Based Constrained delegations rely on that extensions. A requirement to be able to use S4U2proxy is to use an additional service ticket as evidence (usually issued by after S4U2self request). That ticket needs to have the forwardable flag set. There are a few reasons why that flag wouldn't be set on a ticket.

- the "impersonated" user was member of the "Protected Users" group or was configured as "sensitive for delegation"
- the service account configured for constrained delegation was configured for Kerberos only/without protocol transition

In 2020, the "bronze bit" (CVE-2020-17049) was released, allowing attackers to edit a ticket and set the forwardable flag.
## Exploit
The Impacket script getST (Python) can perform all the necessary steps to edit a ticket's flags and obtain a ticket through S4U2proxy to act as another user on a target service (in this case, "Administrator" is impersonated/delegated account but it can be any user in the environment).

The input credentials are those of the compromised service account configured for constrained delegations
```bash
getST.py -force-forwardable -spn "$Target_SPN" -impersonate "Administrator" -dc-ip "$DC_HOST" -hashes :"$NT_HASH" "$DOMAIN"/"$USER"
```
