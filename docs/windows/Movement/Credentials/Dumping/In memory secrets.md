## Theory
Just like the LSASS process on Windows systems allowing for LSASS dumping, some programs sometimes handle credentials in the memory allocated to their processes, sometimes allowing attackers to dump them.
## Exploit
!!! note
    technique needs the attacker to have admin access on the target machine since it involves dumping and handling volatile memory.
=== "UNIX-like"
    On UNIX-like systems, tools like [mimipenguin](https://github.com/huntergregal/mimipenguin) (C, Shell, Python), [mimipy](https://github.com/n1nj4sec/mimipy) (Python) and [LaZagne](https://github.com/AlessandroZ/LaZagne) (Python) can be used to extract passwords from memory.
    ```bash
    mimipenguin
    laZagne memory
    ```
=== "Windows"
    On Windows systems, tools like [LaZagne](https://github.com/AlessandroZ/LaZagne) (Python) and [mimikatz](https://github.com/ParrotSec/mimikatz) (C) can be used to extract passwords from memory but they focus on LSASS dumping.