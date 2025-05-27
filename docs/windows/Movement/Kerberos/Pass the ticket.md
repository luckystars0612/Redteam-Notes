## Theory
There are ways to come across (cached Kerberos tickets) or forge (overpass the hash, silver ticket and golden ticket attacks) Kerberos tickets. A ticket can then be used to authenticate to a system using Kerberos without knowing any password. This is called Pass the ticket. Another name for this is Pass the Cache (when using tickets from, or found on, UNIX-like systems).
## Exploit
!!! note
    **convert tickets UNIX <-> Windows**

    Using ticketConverter.py (from the Impacket suite, written in Python).
    ```bash
    # Windows -> UNIX
    ticketConverter.py $ticket.kirbi $ticket.ccache

    # UNIX -> Windows
    ticketConverter.py $ticket.ccache $ticket.kirbi
    ```
### Inject ticket
=== "UNIX-like"
    Once a ticket is obtained/created, it needs to be referenced in the `KRB5CCNAME` environment variable for it to be used by others tools.
    ```bash
    export KRB5CCNAME=$path_to_ticket.ccache
    ```
=== "Windows"
    The most simple way of injecting the ticket is to supply the /ptt flag directly to the command used to request/create a ticket. Both mimikatz and Rubeus accept this flag.

    This can also be done manually with mimikatz using `kerberos::ptt` or Rubeus.
    ```bash
    # use a .kirbi file
    kerberos::ptt $ticket_kirbi_file

    # use a .ccache file
    kerberos::ptt $ticket_ccache_file
    ```
### Passing the ticket
- On Windows, once Kerberos tickets are injected, they can be used natively.
- On UNIX-like systems, once the KRB5CCNAME variable is exported, the ticket can be used by tools that support Kerberos authentication.
### Modify SPN
When requesting access to a service, a Service Ticket is used. It contains enough information about the user to allow the destination service to decide to grant access or not, without asking the Domain Controller. These information are stored in a protected blob inside the ST called PAC (Privilege Attribute Certificate). In theory, the user requesting access can't tamper with that PAC.

Another information stored in the ST, outside of the PAC, and unprotected, called `sname`, indicates what service the ticket is destined to be used for. This information is basically the SPN (Service Principal Name) of the target service. It's split into two elements: the service class, and the hostname.

Their are multiple service classes for multiple service types (LDAP, CIFS, HTTP and so on). The problem here is that since the SPN is not protected, there are scenarios (e.g. services configured for constrained delegations) where the service class can be modified in the ticket, allowing attackers to have access to other types of services.
=== "UNIX-like"
    This technique is implemented and attempted by default in all Impacket scripts when doing pass-the-ticket (Impacket tries to change the service class to something else, and calls this "AnySPN").

    Impacket's tgssub.py script can also be used for manual manipulation of the service name value
    ```bash
    tgssub.py -in ticket.ccache -out newticket.ccache -altservice "cifs/target"
    ```
=== "Windows"
    With Rubeus, it can be conducted by supplying the `/altservice` flag when using the `s4u` or the `tgssub` modules and the whole SPN can be changed (service class and/or hostname).
    ```bash
    Rubeus.exe tgssub /altservice:cifs /ticket:"base64 | ticket.kirbi"
    ```