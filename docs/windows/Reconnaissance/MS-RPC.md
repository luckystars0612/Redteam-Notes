## Theory
MS-RPC (Microsoft Remote Procedure Call) is a protocol that allows requesting service from a program on another computer without having to understand the details of that computer's network. An MS-RPC service can be accessed through different transport protocols, among which:

- a network SMB pipe (listening ports are 139 & 445)
- plain TCP or plain UDP (listening port set at the service creation)
- a local SMB pipe

RPC services over an SMB transport, i.e. port 445/TCP, are reachable through "named pipes"' (through the IPC$ share). There are many interesting named pipes that allow various operations from NULL sessions context, to local administrative context.

- `\pipe\lsarpc`: enumerate privileges, trust relationships, SIDs, policies and more through the LSA (Local Security Authority)
- `\pipe\samr`: enumerate domain users, groups and more through the local SAM database (only works pre Win 10 Anniversary)
- `\pipe\svcctl`: remotely create, start and stop services to execute commands (used by Impacket's psexec.py and smbexec.py)
- `\pipe\atsvc`: remotely create scheduled tasks to execute commands (used by Impacket's atexec.py)
- `\pipe\epmapper`: used by DCOM (Distributed Component Object Model), itself used by WMI (Windows Management Instrumentation), itself abused by attackers for command execution (used by Impacket's wmiexec.py). DCOM is also used by MMC (Microsoft Management Console), itself abused by attackers for command execution (Impacket's dcomexec.py)
## Find exposed services
The epmapper (MS-RPC EndPoint Mapper) maps services to ports. It uses port 135/TCP and/or port 593/TCP (for RPC over HTTP). Through epmapper, tools like Impacket's [rpcdump.py](https://github.com/un33k/impacket/blob/master/examples/rpcdump.py) (Python) or rpcdump.exe (C) from rpctools can find exposed RPC services
```bash
# with rpcdump.py (example with target port 135/TCP)
rpcdump.py -port 135 $TARGET_IP

# with rpcdump.exe (example with target port 593/TCP)
rpcdump.exe -p 593 $TARGET_IP
```