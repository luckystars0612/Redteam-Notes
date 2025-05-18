The Perl script [enum4linux.pl](https://github.com/CiscoCXSecurity/enum4linux) is a powerful tool able to operate recon techniques for LDAP, NBT-NS and MS-RPC. It's an alternative to a similar program named enum.exe (C++) created for Windows systems. Lately, a rewrite of enum4linux in Python has surfaced, called [enum4linux-ng.py](https://github.com/cddmp/enum4linux-ng). The enum4linux scripts are mainly wrappers around the Samba tools nmblookup, net, rpcclient and smbclient.

```bash
enum4linux-ng.py -A $TARGET_IP
```