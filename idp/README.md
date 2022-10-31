# Configurazione Identity Provider interno

L'IDP interno va configurato in modo da supportare un metodo di autenticazione aggiuntivo a quello usato normalmente
dall'ente.

Di seguito vengono quindi illustrati i passi necessari per la sola delega dell'autenticazione agli IDP SPID.

Il meccanismo usato sfrutta il flusso _multi-factor-authentication_ di SPID per delegare l'autenticazione ad una risorsa
esterna.

I passi da effettuare sono i seguenti:

- [Aggiunta del flusso di autenticazione esterna](#aggiunta-del-flusso-di-autenticazione-esterna);
- [Reperimento degli attributi SPID dall'SP interno](#reperimento-degli-attributi-spid-dallsp-interno);
- [Rilascio degli attributi SPID](#rilascio-degli-attributi-spid);
- [Riconciliazione degli attributi](#riconciliazione-degli-attributi)
- [Rimozione schermata di consenso da IDPint](#rimozione-schermata-di-consenso-da-idpint)

## Aggiunta del flusso di autenticazione esterna

Shibboleth permette di lanciare degli eventi dal form di login, i quali possono essere catturati per differenziare il
flusso di autenticazione da seguire. In questo caso, possiamo definire l'evento `SPID_auth` come trigger per la delega
dell'autenticazione al flusso `RemoteUserInternal`, che a sua volta avvia il flusso di autenticazione presso gli IDP
SPID.

- Modifica della configurazione _MFA_ (file `$IDP_HOME/conf/authn/mfa-authn-config.xml`)

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:util="http://www.springframework.org/schema/util"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
                           http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd"
                           
       default-init-method="initialize"
       default-destroy-method="destroy">
    
<util:map id="shibboleth.authn.MFA.TransitionMap">

	<entry key="">
        <bean parent="shibboleth.authn.MFA.Transition" p:nextFlow="authn/Password" />
    </entry>

    <entry key="authn/Password">
        <bean parent="shibboleth.authn.MFA.Transition"> 
            <property name="nextFlowStrategyMap">
                <map>
                    <entry key="SPID_auth" value="authn/RemoteUser" />
                </map>
            </property>
        </bean>
    </entry> 
 
    <entry key="authn/RemoteUser"  >
        <bean parent="shibboleth.authn.MFA.Transition"/>
    </entry>

</util:map>

</beans>
```

- Modifica del flusso _MFA_ (file `$IDP_HOME/conf/authn/authn-events-flow.xml`)

```
    <end-state id="SPID_auth" />

    <global-transitions>
        <transition on="SPID_auth" to="SPID_auth" />
    </global-transitions>
```

- Abilitazione del flusso MFA e RemoteUserInternal sulla configurazione dell'IDP, `$IDP_HOME/conf/idp.properties`

```
# Regular expression matching login flows to enable, e.g. IPAddress|Password
idp.authn.flows=MFA|RemoteUserInternal
```

- E' necessario inoltre configurare anche la parte IDP di IDPint per fare in modo che il software di IDP possa
  recuperare l'header dal quale ricavare il REMOTE_USER (cfr. documentazione authn/RemoteUser); in questo esempio
  l'header HTTP preso in considerazione è "spid_REMOTE_USER". Il file da modificare
  è `$IDP_HOME/edit-webapp/WEB-INF/web.xml` (che potrebbe non esistere):

```
    <!-- Spid auth delegation -->
    <!-- Servlet protected by container used for RemoteUser authentication -->
    <servlet>
        <servlet-name>RemoteUserAuthHandler</servlet-name>
        <servlet-class>net.shibboleth.idp.authn.impl.RemoteUserAuthServlet</servlet-class>
        <init-param>
           <!-- permette di riconoscere gli headers -->
            <param-name>checkHeaders</param-name>
            <param-value>spid_REMOTE_USER</param-value>
           <!-- permette di riconoscere gli headers --> 
        </init-param>
        <load-on-startup>2</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>RemoteUserAuthHandler</servlet-name>
        <url-pattern>/Authn/RemoteUser</url-pattern>
    </servlet-mapping>
```

⚠️ Attenzione! Si ricorda che a valle della modifica di un file all'interno di `$IDP_HOME/edit-webapp`, è necessario
effettuare il build dell'applicazione IDP mediante il lancio del comando `$IDP_HOME/bin/build.sh` e successivo riavvio
del container.

## Reperimento degli attributi SPID dall'SP interno

A valle dell'autenticazione su SPID, gli attributi SPID sono disponibili negli headers della richiesta. Per leggerli e
renderli visibili come attributi all'IDP interno è possibile definire uno `ScriptedAttribute` all'interno del file
`$IDP_HOME/attribute-resolver.xml` nella forma:

```
    <AttributeDefinition id="spid_attr" xsi:type="ScriptedAttribute" customObjectRef="shibboleth.HttpServletRequest">
                <AttributeEncoder xsi:type="SAML2String" name="spid_attr" friendlyName="spid_attr" />
                <Script><![CDATA[
                if (custom.getHeader("spid_attr_header") != ""  && custom.getHeader("spid_attr") != "(null)" ) {
                        spid_attr.addValue(custom.getHeader("spid_attr"));
                }
        ]]></Script>
    </AttributeDefinition>
```

dove `spid_attr_header` è il nome dell'header da cui leggere l'attributo e `spid_attr` è il nome da dare all'attributo
all'interno dell'IDP interno. Si allega un esempio per l'accesso a tutti gli attuali attributi SPID.

_Nota_: nell'esempio, i nomi degli headers e degli attributi coincidono.

## Rilascio degli attributi SPID

Gli attributi risolti devono poi essere rilasciati secondo le policy opportune, mediante definizione del file
`$IDP_HOME/conf/attribute-filter.xml`. Un esempio di tale policy è presente all'interno del file 
[conf/attribute-filter.xml](conf/attribute-filter.xml).

## Riconciliazione degli attributi

Nel caso in cui l'utente che effettua il login con SPID è presente anche sulla directory delle utenze interne
(per esempio LDAP, o DB) può essere desiderabile effettuare la riconciliazione delle utenze per fondere gli attributi
provenienti da SPID con quelli provenienti dalla directory.

Il modo in cui questo va fatto è strettamente legato al sistema di persistenza delle utenze usato dall'ente. A titolo
esemplificativo, si riporta una modalità che permette di riconciliare l'utenza da LDAP sulla base di un diverso
attributo, a seconda del metodo usato per l'autenticazione:

- _username_, nel caso di autenticazione tramite username e password;
- _codice fiscale_, nel caso di autenticazione tramite SPID.

```
     <DataConnector id="myLDAP"
        xsi:type="LDAPDirectory"
        baseDN="%{ldap.baseDN}"
        principal="%{ldap.principal}"
        principalCredential="%{ldap.principalCredential}"
        trustFile="%{idp.authn.LDAP.trustCertificates}"
        ldapURL="ldaps://%{ldap.hostname}">
        <InputAttributeDefinition ref="spid_fiscalNumber" />
        <FilterTemplate>
            <![CDATA[
                 (|(uid=$resolutionContext.principal), (codFiscale=$spid_REMOTE_USER.get(0).substring(6, 22)), (codFiscale=$spid_fiscalNumber.get(0).substring(6, 22)))
            ]]>
        </FilterTemplate>
    </DataConnector>
```

__Nota__: il codice fiscale per la ricerca su LDAP viene ottenuto come `$spid_REMOTE_USER.get(0).substring(6, 22))`
e `$spid_fiscalNumber.get(0).substring(6, 22))`, poichè il codice fiscale fornito da SPID è nella
forma `TINIT-XXXXXXXXXXXXXXXX` e pertanto i primi 6 caratteri vanno scartati, mantenendo gli ultimi 16.

## Rimozione schermata di consenso da IDPint

⚠️ Nota: Come impostazione predefinita, Shibboleth IDP mostra una schermata per richiedere all'utente il rilascio del consenso
per il transito degli attributi dall'IDP all'SP.

Questo comportamento risulta superfluo, in quanto gli IDP di SPID mostrano già tale schermata e pertanto IDPint non
necessita di ripeterla.

A tale scopo è sufficiente rimuovere dal file `$IDP_HOME/conf/relying-party.xml`, l'attributo
`p:postAuthenticationFlows="attribute-release"` dall'elemento `<bean parent="SAML2.SSO"/>` nel bean
`shibboleth.DefaultRelyingParty`.

```
...

    <bean id="shibboleth.DefaultRelyingParty" parent="RelyingParty">
        <property name="profileConfigurations">
            <list>
		<!-- SAML 1.1 and SAML 2.0 AttributeQuery are disabled by default. -->
		<!--
		<bean parent="Shibboleth.SSO" p:postAuthenticationFlows="attribute-release" />
                <ref bean="SAML1.AttributeQuery" />
                <ref bean="SAML1.ArtifactResolution" />
		<bean parent="SAML2.SSO"/>
                <ref bean="SAML2.ECP" />
                <ref bean="SAML2.Logout" />
                <!--
                <ref bean="SAML2.AttributeQuery" />
                -->
                <ref bean="SAML2.ArtifactResolution" />
                <ref bean="Liberty.SSOS" />
            </list>
        </property>
    </bean>
    
...

```
