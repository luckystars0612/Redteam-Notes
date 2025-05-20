## Theory
BloodHound (Javascript webapp, compiled with Electron, uses Neo4j as graph DBMS) is an awesome tool that allows mapping of relationships within Active Directory environments. It mostly uses Windows API functions and LDAP namespace functions to collect data from domain controllers and domain-joined Windows systems.

## bloodhounce
### Installation
For easily setup, I use [bloodhound community](https://github.com/SpecterOps/BloodHound) version. Download docker-compose file from this [original repo](https://github.com/SpecterOps/BloodHound/blob/main/examples/docker-compose/docker-compose.yml) then just run by docker
```bash
docker compose up
```
### Data collection
There are multiple data collector can be fed into **bloodhoundce**

***Note: bloodhoundce only works with data from community collector***
=== "sharphound"
    [sharphound collector](https://github.com/SpecterOps/SharpHound)

    ```bash
    SharpHound.exe -c All
    ```

    ```bash
    Import-Module .\SharpHound.ps1
    Invoke-BloodHound -CollectionMethod All -OutputDirectory .
    ```
=== "rusthoundce"
    [rusthoundce collector](https://github.com/g0h4n/RustHound-CE)

    ```bash
    rusthound-ce -u user_name -p pass_word -d domain_name -i dc_ip -c All -z
    ```
=== "bloodhound-ce-python"
    [bloodhoun-ce-python collector](https://github.com/dirkjanm/BloodHound.py/tree/bloodhound-ce)

    ```bash
    bloodhound-ce-python -d domain_name -c All -u user_name --hashes ntlm_hash -dc dc_hostname -ns dc_ip --zip
    ```
