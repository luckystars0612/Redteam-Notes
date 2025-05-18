# Port scanning
In an AD environment, there are a plenty of port may open, then we can hunt for DC. The following are some common open ports

- 53/TCP and 53/UDP for DNS
- 88/TCP for Kerberos authentication
- 135/TCP and 135/UDP MS-RPC epmapper (EndPoint Mapper)
- 137/TCP and 137/UDP for NBT-NS
- 138/UDP for NetBIOS datagram service
- 139/TCP for NetBIOS session service
- 389/TCP for LDAP
- 636/TCP for LDAPS (LDAP over TLS/SSL)
- 445/TCP and 445/UDP for SMB
- 464/TCP and 445/UDP for Kerberos password change
- 3268/TCP for LDAP Global Catalog
- 3269/TCP for LDAP Global Catalog over TLS/SSL

## nmap
Nmap can be used to enum open ports

```bash
# -sS       for TCP SYN scan
# -n        for no name resolution
# --open    to only show (possibly) open port(s)
# -p        for port(s) number(s) to scan

nmap -sS -n --open -p 88,389 $IP_RANGE
```

```bash
# -sC	Runs default scripts (equivalent to --script=default). Useful for basic enumeration.
# -sV	Attempts to detect service versions on open ports.
# -Pn	Skips host discovery (assumes all hosts are up). Useful in networks where ICMP is blocked.
# -T4	Sets timing template to 4 (aggressive). Speeds up the scan, but might be noisy.

$IP_RANGE	The target IP address or range (e.g., 192.168.1.0/24, 10.10.11.23).
nmap -sC -sV -Pn -T4 $IP_RANGE
```