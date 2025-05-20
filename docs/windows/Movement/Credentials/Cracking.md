## Hashcat
List of useful hash types in AD environment

| **Hash Type**                         | **-m / --hash-type Number** |
|--------------------------------------|------------------------------|
| LM hash                              | 3000                         |
| NT hash                              | 1000                         |
| LM response                          | not supported                |
| LMv2 response                        | not supported                |
| NTLM response                        | 5500                         |
| NTLMv2 response                      | 5600                         |
| (DCC1) Domain Cached Credentials     | 1100                         |
| (DCC2) Domain Cached Credentials 2   | 2100                         |
| ASREQroast                           | 7500                         |
| ASREProast                           | 18200                        |
| Kerberoast                           | 13100                        |

### Dictionary attack
```bash
hashcat --attack-mode 0 --hash-type $number $hashes_file $wordlist_file
```
### Bruteforce attack
```bash
hashcat --attack-mode 3 --increment --increment-min 4 --increment-max 8 --hash-type $number $hashes_file "?a?a?a?a?a?a?a?a?a?a?a?a"
#hashcat command that bruteforces any password from 4 to 8 characters long. Each character can be any printable character.
```
Hashcat has the following built-in charsets that can be used.
```bash
?l = abcdefghijklmnopqrstuvwxyz
?u = ABCDEFGHIJKLMNOPQRSTUVWXYZ
?d = 0123456789
?s = !"#$%&'()*+,-./:;<=>?@[\]^_`{|}~
?a = ?l?u?d?s
?b = 0x00 - 0xff
```
Below are examples of hashcat being used with built-in charset.
```bash
# Passwords are like : 1 capital letter, 3 letters, 4 numbers, 1 special char
hashcat --attack-mode 3 --hash-type $number $hashes_file "?u?l?l?l?d?d?d?d?s"

# Password are 8 chars-long and can be any printable char.
hashcat --attack-mode 3 --hash-type $number $hashes_file "?a?a?a?a?a?a?a?a"
```
Hashcat can also be started with custom charsets in the following manner.
```bash
hashcat --attack-mode 3 --custom-charset1 "?u" --custom-charset2 "?l?u?d" --custom-charset3 "?d" --hash-type $number $hashes_file "?1?2?2?2?3"
```
Hashcat also has an incremental feature that allows to bruteforce passwords up to a certain length whereas the commands above only try the specified mask's length.
```bash
# Password are up to 8 chars-long and can be any printable char.
hashcat --attack-mode 3 --increment --hash-type $number $hashes_file "?a?a?a?a?a?a?a?a"

# Password are 4 to 8 chars-long and can be any printable char (mask length is 12 so that --increment-max can be upped to 12).
hashcat --attack-mode 3 --increment --increment-min 4 --increment-max 8 --hash-type $number $hashes_file "?a?a?a?a?a?a?a?a?a?a?a?a"
```
## JohntheRipper
A robust alternative to hashcat is John the Ripper, a.k.a. john (C). It handles some hash types that hashcat doesn't (Domain Cached Credentials for instance) but it also has a strong community that regularly releases tools in the form of "something2john" that convert things to a john crackable format (e.g. `bitlocker2john, 1password2john, keepass2john, lastpass2john` and so on).
