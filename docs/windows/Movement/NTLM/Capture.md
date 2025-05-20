[Responder](https://github.com/SpiderLabs/Responder) (Python) and [Inveigh](https://github.com/Kevin-Robertson/Inveigh) (Powershell) are great tools able to do name poisoning for forced authentication attacks, but also able to capture responses (LM or NTLM hashes) by starting servers waiting for incoming authentications. Once those listening servers are up and ready, the tester can initiate the forced authentication attack.
=== "Responder"
    From UNIX-like systems, Responder (Python) can be used to start servers listening for NTLM authentications over many protocols (SMB, HTTP, LDAP, FTP, POP3, IMAP, SMTP, ...). Depending on the authenticating principal's configuration, the NTLM authentication can sometimes be downgraded with `--lm` and `--disable-ess` in order to obtain NTLMv1 responses.
    ```bash
    responder --interface "eth0" --analyze
    responder -I "eth0" -A

    # with downgrading
    responder --interface "eth0" --analyze --lm --disable-ess
    ```
    Testers should try to force a LM hashing downgrade with Responder. LM and NTLMv1 responses (a.k.a. LM/NTLMv1 hashes) from Responder can easily be cracked with [crack.sh](https://crack.sh/netntlm/). The [ntlmv1-multi](https://github.com/evilmog/ntlmv1-multi) tool (Python) can be used to convert captured responses to crackable formats by hashcat, crack.sh and so on.
    ```bash
    ntlmv1-multi --ntlmv1 SV01$::BREAKING.BAD:AD1235DEAC142CD5FC2D123ADCF51A111ADF45C2345ADCF5:AD1235DEAC142CD5FC2D123ADCF51A111ADF45C2345ADCF5:1122334455667788
    ```
    !!! note
        Machine account NT hashes can be used with the `Silver Ticket` or `S4U2self` abuse techniques to gain admin access to it.
=== "Inveigh"
    Start poisoning LLMNR, NBT-NS and mDNS with a custom challenge, enable HTTPS capturing, enable proxy server authentication captures
    ```bash
    .\Inveigh.exe -Challenge 1122334455667788 -ConsoleOutput Y -LLMNR Y -NBNS Y -mDNS Y -HTTPS Y -Proxy Y
    ```