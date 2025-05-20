## Theory
Theory
There are scenarios where testers need to operate credential guessing for lateral movement, privilege escalation, or for obtaining a foothold on a system. Depending on the target user (or target system), guessing can be achieved differently.

There are three major types of credential guessing.

- Common passwords: a list of common passwords (admin, backup, qwerty, ...) are tested against one or more users. This technique is to be used with care since it can cause lockouts and a lot of noise, depending on the target organization's policies.
- Default passwords: certain technologies come with default passwords that are often left unchanged. This technique is similar to the "common passwords" technique, but generates less noise and has fewer chances of locking out an account.
- Situational guessing: there are many cases where user passwords are not common and yet easily guessable when knowing the organization's vocabulary and context.
## Exploit
### Common passwords
Depending on the target service, different lists of common passwords (e.g. [SecLists](https://github.com/danielmiessler/SecLists)) and different tools can be used.

- [Hydra](https://www.kali.org/tools/hydra/) (C) can be used against a lot (50+) of services like FTP, HTTP, IMAP, LDAP, MS-SQL, MYSQL, RDP, SMB, SSH and many many more.
- [NetExec](https://github.com/Pennyw0rth/NetExec) (Python) can be used against LDAP, WinRM, SMB, SSH and MS-SQL.
- [Kerbrute](https://github.com/ropnop/kerbrute) (Go) and [smartbrute](https://github.com/ShutdownRepo/smartbrute) (Python) can be used against Kerberos pre-authentication.
!!! note
    netexec has useful options for password guessing

    - --no-bruteforce: tries user1 -> password1, user2 -> password2 instead of trying every password for every user
    - --continue-on-success: continues authentication attempts even after successes

    Smartbrute has equivalent features

    - --line-per-line: equivalent of netexec's --no-bruteforce option
    - --stop-on-success: reversed equivalent of netexec's --continue-on-success option

### Default password
- [SecLists's Default Credentials Sheet](https://github.com/danielmiessler/SecLists/blob/master/Passwords/Default-Credentials/default-passwords.csv) (for password)
- [ihebski's DefaultCreds cheatsheet](https://github.com/ihebski/DefaultCreds-cheat-sheet) (for passwords and SSH keys)

### Bruteforcing with lockout policies
[Smartbrute](https://github.com/ShutdownRepo/smartbrute) (Python) and [sprayhound](https://github.com/Hackndo/sprayhound) (Python) can dynamically fetch the organization's users and lockout policy to only bruteforce accounts that have a few attempts left in order to avoid locking them out. This can only be done when supplying a valid Active Directory account's credentials. These tools have the ability to check if accounts have their username set as password. Finally, a great additional feature is that neo4j is supported and compromised accounts can be set as owned (useful when working with BloodHound).