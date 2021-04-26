# Configurazione Apache

In questo documento vengono riportate esclusivamente le istruzioni specifiche per la configurazione del proxy. Questo 
non include la configurazione necessaria per le seguenti attività:

- Proxying delle richieste destinate all'idp verso l'application server (Jetty/Tomcat). Per quest'attività è possibile 
  fare riferimento al [tutorial IDEM](https://github.com/ConsortiumGARR/idem-tutorials/blob/master/idem-fedops/HOWTO-Shibboleth/Identity%20Provider/Debian-Ubuntu/HOWTO%20Install%20and%20Configure%20a%20Shibboleth%20IdP%20v3.4.x%20on%20Debian-Ubuntu%20Linux%20with%20Apache2%20%2B%20Jetty9.md) e alla [documentazione ufficiale di Shibboleth IDP](https://wiki.shibboleth.net/confluence/display/IDP30/Installation);
- Configurazione del DiscoveryService Shibboleth. Per quest'attività è possibile fare riferimento alla 
  [documentazione ufficiale](https://wiki.shibboleth.net/confluence/display/EDS10/Embedded+Discovery+Service).

Apache richiede la configurazione di due endpoint dell'IDP:

- endpoint per l'autenticazione esterna `/idp/Authn/RemoteUser`, al fine di proteggerlo con Shibboleth SP; 
- endpoint per l'acquisizione del profilo `idp/profile`, al fine di consentire il transito degli attributi SPID dall'SP all'IDP mediante RequestHeader.

## `/idp/Authn/RemoteUser`

Questa rotta viene invocata dall'IDP per ottenere informazioni relative al REMOTE_USER. Essa va protetta col modulo 
Shibboleth SP in modo da innescare l'autenticazione SPID. Successivamente il REMOTE_USER viene fatto transitare mediante
creazione di un header HTTP.

```
<Location "/idp/Authn/RemoteUser">
AuthType shibboleth
ShibRequestSetting requireSession true
Require shibboleth

        # La seguente direttiva manda il REMOTE_USER all'IDPint
        RequestHeader unset spid_REMOTE_USER
        RequestHeader set spid_REMOTE_USER expr=%{REMOTE_USER}
 </Location>
```
## `idp/profile`

La configurazione di questa rotta prevede l'abilitazione del modulo Shibboleth SP. Gli attributi sono fatti transitare 
dalla parte SP alla parte IDP mediante meccanismo basato su headers HTTP. 

Ciascun header verrà popolato con i valori degli attributi ottenuti da SPID e potrà essere letto dall'IDP mediante 
la definizione appropriata di uno `ScriptedAttribute`.

Si consiglia di usare nomi degli header non prevedibili per evitare vulnerabilità legate all'iniezione di header 
artefatti. A questo scopo è indispensabile cancellare qualsiasi valore già presente in ciascun header.

_Nota_: l'attributo menzionato all'interno di `%{reqenv:attr_name}` deve coincidere col nome definito per l'attributo 
all'interno di `$SP_HOME/attribute-map.xml`.

_Nota (2)_: è possibile eliminare le parti relative ad attributi SPID non necessari.

```
<Location "/idp/profile">
AuthType shibboleth
ShibRequestSetting requireSession false
Require shibboleth

    # Le righe successive trasportano il valore degli attributi ricevuti da IDPext
    # rendendoli disponibili, come header, a IDPint
    # idpext_attribute1, idpext_attribute2, ecc.ecc. sono i nomi degli attributi, mappati secondo attribute-map.xml dell'SP 
    # 

        RequestHeader unset spid_spidCode
        RequestHeader set spid_spidCode expr=%{reqenv:spidCode}

        RequestHeader unset spid_name
        RequestHeader set spid_name expr=%{reqenv:name}

        RequestHeader unset spid_familyName
        RequestHeader set spid_familyName expr=%{reqenv:familyName}

        RequestHeader unset spid_placeOfBirth
        RequestHeader set spid_placeOfBirth expr=%{reqenv:placeOfBirth}

        RequestHeader unset spid_countyOfBirth
        RequestHeader set spid_countyOfBirth expr=%{reqenv:countyOfBirth}

        RequestHeader unset spid_dateOfBirth
        RequestHeader set spid_dateOfBirth expr=%{reqenv:dateOfBirth}

        RequestHeader unset spid_gender
        RequestHeader set spid_gender expr=%{reqenv:gender}

        RequestHeader unset spid_companyName
        RequestHeader set spid_companyName expr=%{reqenv:companyName}

        RequestHeader unset spid_registeredOffice
        RequestHeader set spid_registeredOffice expr=%{reqenv:registeredOffice}

        RequestHeader unset spid_fiscalNumber
        RequestHeader set spid_fiscalNumber expr=%{reqenv:fiscalNumber}

        RequestHeader unset spid_ivaCode
        RequestHeader set spid_ivaCode expr=%{reqenv:ivaCode}

        RequestHeader unset spid_idCard
        RequestHeader set spid_idCard expr=%{reqenv:idCard}

        RequestHeader unset spid_mobilePhone
        RequestHeader set spid_mobilePhone expr=%{reqenv:mobilePhone}

        RequestHeader unset spid_email
        RequestHeader set spid_email expr=%{reqenv:email}

        RequestHeader unset spid_address
        RequestHeader set spid_address expr=%{reqenv:address}

        RequestHeader unset spid_expirationDate
        RequestHeader set spid_expirationDate expr=%{reqenv:expirationDate}

        RequestHeader unset spid_digitalAddress
        RequestHeader set spid_digitalAddress expr=%{reqenv:digitalAddress}

</Location>
```
