## smbclient
Logon into smb shares
```bash
smbclient //10.10.11.69/IT -U j.fleischman
```
Download files
```bash
get file_name
```
## nxc
Get user through rid-brute nxc
```bash
nxc smb fluffy.htb -u 'j.fleischman' -p 'J0elTHEM4n1990!' --rid-brute | grep "SidTypeUser" | awk -F '\\' '{print $2}' | awk '{print $1}' > users.txt
```