# Java Naming and Directory Interface (JNDI)

1. [Was ist das JNDI und wofür wird es verwendet?](#was-ist-das-jndi-und-wofür-wird-es-verwendet)
2. [Welche Rolle übernimmt das JNDI auf einem Application Server?](#welche-rolle-übernimmt-das-jndi-auf-einem-application-server)
3. [Was sind Namensräume? Welche Namensräume gibt es?](#was-sind-namensräume-welche-namensräume-gibt-es)
4. [Wie funktioniert Resource Injection?](#wie-funktioniert-resource-injection)
   1. [Wie werden lokale Ressourcen definiert?](#wie-werden-lokale-ressourcen-definiert)
   2. [Was bedeutet der `AuthenticationType` (`<res-auth>`)?](#was-bedeutet-der-authenticationtype-res-auth)
   3. [Was ist bei der `@Resource`-Annotation der Unterschied zwischen `name`, `lookup` und `mappedName`](#was-ist-bei-der-resource-annotation-der-unterschied-zwischen-name-lookup-und-mappedname)

## Was ist das JNDI und wofür wird es verwendet?

JNDI (Java Naming and Directory Interface) ist eine Java-API zum standardisierten Zugriff auf LDAP- und Namensdienste (z.B. DNS, RMI Registry). 

Hier ein Beispiel für den Zugriff auf einen LDAP-Server mit JNDI:

```java
import javax.naming.Context;
import javax.naming.NamingEnumeration;
import javax.naming.NamingException;
import javax.naming.directory.*;

import java.util.Hashtable;

public class JndiLdapExample {

  public static void main(String[] args) {
    // LDAP-Server-URL (z. B. Active Directory oder OpenLDAP)
    final var ldapUrl = "ldap://example.com:389";
    final var baseDn = "dc=example,dc=com";
    final var searchFilter = "(uid=john.doe)";
    // Admin-Benutzer für Authentifizierung
    final var bindDn = "cn=admin,dc=example,dc=com";
    final var bindPassword = "password";

    // JNDI-Umgebungseinstellungen für die LDAP-Verbindung
    final Hashtable<String, String> env = new Hashtable<>();
    env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.ldap.LdapCtxFactory");
    env.put(Context.PROVIDER_URL, ldapUrl);
    env.put(Context.SECURITY_AUTHENTICATION, "simple");
    env.put(Context.SECURITY_PRINCIPAL, bindDn);
    env.put(Context.SECURITY_CREDENTIALS, bindPassword);

    // Verbindung zum LDAP-Server herstellen
    try(DirContext ctx = new InitialDirContext(env)) {

      // LDAP-Suchkriterien definieren
      var searchControls = new SearchControls();
      searchControls.setSearchScope(SearchControls.SUBTREE_SCOPE); // Rekursive Suche

      // Suche im LDAP-Verzeichnis
      var results = ctx.search(baseDn, searchFilter, searchControls);

      while (results.hasMore()) {
        var result = results.next();
        var attrs = result.getAttributes();
        System.out.println("Gefundener Eintrag: " + result.getNameInNamespace());

        // Alle Attribute ausgeben
        var attributes = attrs.getAll();
        while (attributes.hasMore()) {
          var attr = attributes.next();
          System.out.println(attr.getID() + ": " + attr.get());
        }
      }

    } catch (NamingException e) {
      e.printStackTrace();
    }
  }

}
```

## Welche Rolle übernimmt das JNDI auf einem Application Server?

Auf einem Application Server dient das JNDI als eine Art Telefonbuch für Java-Anwendungen, um Ressourcen wie Datenquellen, EJBs, JMS-Queues/Topics oder andere verteilte Objekte zu lokalisieren.

> [!NOTE]
> Bei Datenbankverbindungen werden Treiber, URL, Benutzername und Passwort am Application Server konfiguriert. Der Application Server verwaltet die Verbindungen (Connection Pooling). In der Anwendung erhalten wir eine Referenz auf eine Verbindung über einen _JNDI-Lookup_ bzw. über _Resource Injection_ (empfohlen).

## Was sind Namensräume? Welche Namensräume gibt es?

Ein JNDI-Namensraum ist eine hierarchische Struktur, in der Ressourcen über logische Namen referenziert werden. Diese Struktur ermöglicht es, Ressourcen unabhängig vom Code zu verwalten und zu konfigurieren.

JNDI stellt mehrere standardisierte Namensräume bereit, die für unterschiedliche Zwecke in Java-/Jakarta-EE-Anwendungen verwendet werden.

### Java-/Jakarta-EE Standard-Namensräume

| Namensraum      | Beispiel                   | Anwendungsfälle                                                         |
|-----------------|----------------------------|-------------------------------------------------------------------------|
| `java:comp/env` | `java:comp/env/jdbc/myDB`  | Anwendungsspezifische Ressourcen, in Deployment-Deskriptor konfiguriert |
| `java:comp`     | `java:comp/MyBean`         | Komponentenspezifisch, z.B. lokale EJBs                                 |
| `java:module`   | `java:module/MyBean`       | Modulweit - z.B. EJBs innerhalb eines Moduls (Web-/EJB-Modul)           |
| `java:app`      | `java:app/MyBean`          | Anwendungsweit - EJBs innerhalb einer Anwendung (EAR)                   |
| `java:global`   | `java:global/MyApp/MyBean` | Anwendungsübergreifend - globaler Lookup                                |

## Wie funktioniert Resource Injection?

Resource Injection ist eine Technik in Java/Jakarta EE, mit der Abhängigkeiten (z.B. Datenbankverbindungen, EJBs, JMS-Warteschlangen) automatisch durch den Application Server bereitgestellt werden, ohne dass der Entwickler sie manuell über JNDI suchen muss.

Die Annotation `@Resource` sucht im `java:comp/env`-Namensraum nach einer konfigurierten Datenbankverbindung o.a. Ressourcen:

```java
@ApplicationScoped
public class DatabaseService { 

  @Resource(name = "jdbc/myDB") // Injection der DataSource
  DataSource dataSource;

  public void getData() throws SQLException {
    try (Connection conn = dataSource.getConnection()) {
      System.out.println("Verbindung erfolgreich: " + conn);
    }
  }

}
```

Für EJBs gibt es eine eigene Annotation:

```java
@Stateless
public class MyService {

    // ...

}

@WebServlet
public class MyServlet extends HttpServlet {

    @EJB
    private MyService myService;

    // ...
}
```

### Wie werden lokale Ressourcen definiert?

Ressourcen wie z.B. Datenbankverbindungen werden am Server konfiguriert und dann im globalen Namensraum des JNDI bereitgestellt. Vor allem bei Application Servern, auf denen mehrere Anwendungen laufen, empfiehlt sich die Definition eines lokalen Ressourcennamens, um Konflikte zwischen den Anwendungen zu vermeiden. Hierfür sind folgende Schritte notwendig:

Die anwendungsspezifischen Ressourcen werden namentlich in den Deployment-Deskriptoren (`web.xml` und `ejb-jar.xml`) festgelegt.

```xml
<resource-ref>
  <res-ref-name>jdbc/myDB</res-ref-name>
  <res-type>javax.sql.DataSource</res-type>
  <res-auth>Container</res-auth>
</resource-ref>
```

> [!TIP]
> Alternativ kann dies auch per Annotation geschehen. Somit kann die Notwendigkeit der `web.xml` bzw. `ejb-jar.xml` vermieden werden. Dies eignet sich v.a. für Ressourcen, die von genau einer Klasse benötigt werden.

```java
@Resource(
  name = "jdbc/myDB", 
  type = DataSource.class,
  authenticationType = Resource.AuthenticationType.CONTAINER // default
  )
@ApplicationScoped
public class MyService {
  
  // Resource Injection oder JNDI-Lookup
  
}
```

Beim Deployment werden diese lokalen (anwendungsinternen) Namen durch gesonderte, server-spezifische Konfiguration auf die globalen Ressourcen gemappt. Beim Liberty erfolgt dies z.B. mit einem `<resource-ref>`-Element:

```xml
<server>
  <!-- globale Ressource -->
  <dataSource id="MyGlobalDataSource" jndiName="jdbc/globalDB">
    <jdbcDriver libraryRef="DerbyLib"/>
    <properties.derby databaseName="sample" user="dbuser" password="dbpass"/>
  </dataSource>
  <!-- Anwendung mit Mapping auf lokalen Ressourcennnamen -->
  <application id="myApp" location="myApp.war" name="myApp">
    <resourceRef name="jdbc/myDB" bindingName="jdbc/globalDB"/>
  </application>
</server>
```

### Was bedeutet der `AuthenticationType` (`<res-auth>`)?

Der `AuthenticationType` gibt an, wer die Verantwortlichkeit zur Authentifizierung bei der Ressource (z.B. Datenbank) trägt. Möglich sind 2 verschiedene Werte: 

- `CONTAINER`: Der Application Server übernimmt die Authentifizierung. Die Credentials sind am Server hinterlegt.
- `APPLICATION`: Die Anwendung übernimmt die Authentifizierung. Die Credentials werden in der Anwendung ermittelt.

Der Unterschied im Code wird dann beim Zugriff auf die Datenbank deutlich:

```java
@ApplicationScoped
public class MyService {
  
  @Resource(name = "jdbc/myDB")
  DataSource ds; 
  
  public void doDatabaseAccess() {
    // AuthType: CONTAINER
    try (Connection con = ds.getConnection()) {
      // ...
    }
    // AuthType: APPLICATION
    try (Connection con = ds.getConnection("user", "password")) {
      // ...
    }
  }
  
}
```

> [!CAUTION]
> `APPLICATION` sollte mit Bedacht verwendet werden, weil Credentials durch die Anwendung gehen. Die üblichen Fallstricke (Logging der Credentials, Ablegen der Credentials im Sourcecode) müssen vermieden werden. Letztlich benötigen wir diese Einstellung nur dann, wenn die Credentials dynamisch zur Laufzeit ermittelt werden müssen.

### Was ist bei der `@Resource`-Annotation der Unterschied zwischen `name`, `lookup` und `mappedName`?

Diese drei Attribute der `@Resource`-Annotation führen oft zu Verwirrung, weil sie ähnlich klingen – aber sie haben unterschiedliche Bedeutungen und Anwendungsfälle:

| Attribut     | Bedeutung                                             | Verwendung                                                                                              |
|--------------|-------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `name`       | JNDI-Name im lokalen Namensraum (`java:comp/env/...`) | Lokale Referenz innerhalb der Anwendung, die mit `<resource-ref>` gemappt ist.                          |
| `lookup`     | Vollständiger JNDI-Name (direktes Lookup)             | Empfohlen, wenn kein `<resource-ref>` genutzt wird                                                      |
| `mappedName` | Globaler JNDI-Name (server-weite Resource)            | Direktes Mapping auf eine globale Resource (container-spezifisch, nicht portabel) - **nicht empfohlen** |

> [!TIP]
> Seit Java EE 6 sollte für Lookups im globalen Namensraum `lookup` verwendet werden. `mappedName` wird in älteren Anwendungen verwendet, ist aber nicht standardisiert.

```java
@ApplicationScoped
public class MyService {

  // lokaler Namensraum, mit <resource-ref>
  @Resource(name = "jdbc/myDB") // java:comp/env/jdbc/myDB
  DataSource ds; 

  // lokaler Namensraum, mit <resource-ref>
  @Resource(lookup = "java:global/jdbc/globalDB") // java:comp/env/jdbc/myDB
  DataSource globalDs; 

}
```
