This technique is a form of [pass the key](./Pass%20the%20key.md).

Kerberos offers 4 different key types: DES, RC4, AES-128 and AES-256.

When attackers know the RC4 key (which is in fact the user's NT hash), and when the RC4 etype is not disabled, they can use it to obtain Kerberos tickets.