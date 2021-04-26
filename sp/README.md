# Configurazione Service Provider SPID

La parte SP del proxy consta di un normale SP SPID, per l'implementazione si rimanda al repository 
[italia/spid-sp-shibboleth](https://github.com/italia/spid-sp-shibboleth).

Unica differenza è il discovery service da utilizzare. In questo caso viene usato l'[Embedded Discovery Service di 
Shibboleth](https://wiki.shibboleth.net/confluence/display/EDS10/Embedded+Discovery+Service).