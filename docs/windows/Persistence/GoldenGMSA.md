## Theory
**What is gMMSA account**

Within an Active Directory environment, service accounts are often created and used by different applications. These accounts usually have a password that is rarely updated. To address this issue, it is possible to create Group Managed Service Accounts (gMSA), which are managed directly by AD, with a strong password and a regular password rotation.

The password of a gMSA account can legitimately be requested by authorized applications. In that case, an LDAP request is made to the domain controller, asking for the gMSA account's `msDS-ManagedPassword` attribute's value.

The "GoldenGMSA" persistence lies in the fact that the KDS root keys used for gMSA password calculation don't change (at least not without some admin intervention or custom automation). Once they are exfiltrated and saved, any gMSA account password can be calculated since the additional values needed can be obtained by any low-privileged user.

## Exploit
### Obtain persistence
Once an AD environment is compromised, acquiring the "GoldenGMSA" persistence requires to dump the KDS root keys.
=== "Windows"
    The KDS (Key Distribution Service) root keys can be exfiltrated from the domain with high-privileged access with [GoldenGMSA](https://github.com/Semperis/GoldenGMSA) (C#).

    Without the `--forest` argument, the forest root domain is queried, hence requiring Enterprise Admins or Domain Admins privileges in the forest root domain, or SYSTEM privileges on a forest root Domain Controller.
    ```bash
    GoldenGMSA.exe kdsinfo
    ```
    With the `--forest` argument specifying the target domain or forest, SYSTEM privileges are required on the corresponding domain or forest Domain Controller. In case a child domain is specified, the parent domain keys will be dumped as well.
    ```bash
    GoldenGMSA.exe kdsinfo --forest child.lab.local
    ```
### REtrieving gMSA passwords
Later on, the attacker can then, with low-privileged access to the domain:

- dump some information relative to the gMSA account to retrieve the password for
- use those elements to calculate the gMSA password
#### Account information dump
=== "Windows"
    In addition to the `KDS root` keys, the following information, relative to a gMSA, need to be dumped in order to compute its password:

    - SID (Security IDentifier)
    - RootKeyGuid: indicating what KDS root key to use
    - Password ID: which rotates regularly

    The information can be dumped with low-privilege access to AD with [GoldenGMSA](https://github.com/Semperis/GoldenGMSA) (C#).
    ```bash
    GoldenGMSA.exe gmsainfo
    ```
    In order to dump the necessary information of a single gMSA, its SID can be used as filter with the `--sid` argument.
    ```bash
    GoldenGMSA.exe gmsainfo --sid "S-1-5-21-[...]1586295871-1112"
    ```
#### Password calculation
=== "Windows"
    Given a gMSA SID, the corresponding KDS root key (matching the RootKeyGuid obtained beforehand), and the Password ID, the actual plaintext password can be calculated with GoldenGMSA (C#)
    ```bash
    GoldenGMSA.exe compute --sid "S-1-5-21-[...]1586295871-1112" --kdskey "AQA[...]jG2/M=" --pwdid "AQAAAEtEU[...]gBsAGEAYgBzAAAA"
    ```
    Since the password is randomly generated and is not intended to be used by real users with a keyboard (but instead by servers, programs, scripts, etc.) the password is very long, complex and can include non-printable characters. GoldenGMSA will output the password in base64.

    In order to use the password, its MD4 (i.e. NT) hash can be calculated, for pass the hash.
    ```bash
    import base64
    import hashlib

    b64 = input("Password Base64: ")

    print("NT hash:", hashlib.new("md4", base64.b64decode()).hexdigest())'
    ```
