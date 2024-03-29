# SPID Shibboleth Proxy

Shibboleth IDP con possibilità di delegare l'autenticazione ad un IDP SPID/CIE/eIDAS.

![Schema IDP Proxy](gallery/proxy.png)

⚠️ Attenzione! Questo progetto ha la finalità di esempio di base che può richiedere variazioni di configurazione in base alle specifiche necessità di ciascun ente. Apri nuove issue e raggiungici sul [canale Slack](https://developersitalia.slack.com/archives/C73R3UQE8) per contribuire.

Lo scenario di base prevede che un ente abbia un proprio Identity Provider sul quale voglia implementare la possibilità di fornire autenticazione tramite SPID, CIE o eIDAS.

Il setup dimostrativo contiene i seguenti elementi:

- [Shibboleth IDP](idp)
- [Shibboleth SP](sp)
- [Discovery Service](discovery-service)
- [Configurazione di Apache2 con mod_shib](apache)
