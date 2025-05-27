## Theory
[Golden](./Golden%20tickets.md) and [Silver tickets](./Silver%20tickets.md) can usually be detected by probes that monitor the service ticket requests (KRB_TGS_REQ) that have no corresponding TGT requests (KRB_AS_REQ). Those types of tickets also feature forged PACs that sometimes fail at mimicking real ones, thus increasing their detection rates. Diamond tickets can be a useful alternative in the way they simply request a normal ticket, decrypt the PAC, modify it, recalculate the signatures and encrypt it again. It requires knowledge of the target service long-term key (can be the krbtgt for a TGT, or a target service for a Service Ticket).

Since Golden and Silver tickets are fully forged are not preceded by legitimate TGT (KRB_AS_REQ) or Service Ticket requests (KRB_TGS_REQ), detection rates are quite high. Diamond tickets are an alternative to obtaining similar tickets in a stealthier way.

In this scenario, an attacker that has knowledge of the service long-term key (krbtgt keys in case of a TGT, service account keys of Service Tickets) can request a legitimate ticket, decrypt the PAC, modify it, recalculate the signatures and encrypt the ticket again. This technique allows to produce a PAC that is highly similar to a legitimate one and also produces legitimate requests.

=== "UNIX-like"

    ```bash
    ticketer.py -request -domain "$DOMAIN" -user "$USER" -password "$PASSWORD" -nthash 'krbtgt/service NT hash' -aesKey 'krbtgt/service AES key' -domain-sid 'S-1-5-21-...' -user-id '1337' -groups '512,513,518,519,520' 'baduser'
    ```
=== "Windows"
    From Windows systems, Rubeus (C#) can be used for such purposes
    ```bash
    Rubeus.exe diamond /domain:DOMAIN /user:USER /password:PASSWORD /dc:DOMAIN_CONTROLLER /enctype:AES256 /krbkey:HASH /ticketuser:USERNAME /ticketuserid:USER_ID /groups:GROUP_IDS
    ```