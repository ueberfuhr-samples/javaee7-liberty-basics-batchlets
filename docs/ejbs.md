# Enterprise Java Beans (EJBs)

## Inhalte dieser Seite

1. [Was sind EJBs?](#was-sind-ejbs)
2. [Wofür werden EJBs verwendet?](#wofür-werden-ejbs-verwendet)
3. [Welche Arten von EJBs gibt es?](#welche-arten-von-ejbs-gibt-es)
   1. [Stateless Session Beans](#stateless-session-beans)
   2. [Stateful Session Beans](#stateful-session-beans)
   3. [Message Driven Beans (MDB)](#message-driven-beans-mdb)
   4. [Timer Beans](#timer-beans)
   5. [Startup Beans](#startup-beans)
4. [Wofür gibt es Local und Remote Interfaces?](#wofür-gibt-es-local-und-remote-interfaces)

## Was sind EJBs?

Enterprise JavaBeans (EJB) sind eine Java-basierte Server-Komponente, die speziell für den Einsatz in Enterprise-Anwendungen entwickelt wurde. EJBs helfen, wiederkehrende Aufgaben wie Transaktionsmanagement, Sicherheit, Threading und Remoting zu vereinfachen, indem sie diese an den Application Server delegieren.

## Wofür werden EJBs verwendet?

- Verwalten von [Transaktionen](transactions.md) (JTA, Container Managed Transactions)
- Verteilte Aufrufe (_Remote EJBs_ über RMI/IIOP oder HTTP)
- Thread Pooling, Thread Safety und Clustering, Load Balancing
- Integration in JNDI und andere Java EE Services (z. B. JMS, JPA)

> [!TIP]
> Früher hatten EJBs noch weitere Funktionen wie
> - Dependency Injection von Business-Logik
> - Security (z. B. Rollen, Autorisierung)
> - Auslagern technischer Aspekte über _Interceptors_
>
> Diese wurden in unabhängige Technologien (v.a. CDI) ausgelagert, sodass für diese Zwecke nicht mehr zwingend mit EJBs gearbeitet werden muss. Selbst die Verwaltung von Transaktionen ist in CDI möglich, dann aber mit anderem Lebenszyklus. Für verteilte Architekturen werden mittlerweile REST-Services bevorzugt.

## Welche Arten von EJBs gibt es?

### Stateless Session Beans

- keine Zustände zwischen den Methodenaufrufen
- ideal für Service-Methoden (z. B. Business-Logik ohne Session-Daten)

```java
@Stateless
public class PaymentService {
    public void processPayment(Order order) {
        // ... 
    }
}
```

### Stateful Session Beans

- Speichert den Zustand pro Client (Session⚠)
- für Workflows oder langlaufende Dialoge

```java
@Stateful
public class ShoppingCart implements Serializable {

  private static final long serialVersionUID = 1L;

  private List<Item> items;

  public void addItem(Item i) {
    // ..
  }
}
```

> [!CAUTION]
> Stateful Session Beans müssen serialisierbar sein, d.h. sie müssen in einem Byte-Stream umgewandelt werden können. Java unterstützt dies durch Implementierung des `java.io.Serializable`-Interfaces. Instanzvariablen müssen dann ebenfalls serialisierbar sein.
> 
> Serialisierung wird benötigt für
> - Aktivierung/Passivierung (Auslagern von Objekten aus dem RAM auf Plattenspeicher)
> - Session Replication (zwischen den Knoten im Cluster)
> - Persistente Sessions
> 
> Bei Nichtbeachtung erfolgt ein Fehler (`NotSerializableException`) entweder direkt beim Deployment, oder im schlimmsten Fall erst bei Eintritt eines dieser Ereignisse (in Produktion unter Last).

> [!TIP]
> Instanzvariablen, die nicht serialisiert werden sollen, werden mit `@Transient` annotiert.

```java
@Stateful
public class ShoppingCart implements Serializable {

  // ...
  @Transient
  @Resource
  private SessionContext ctx;
    
}
```

> [!TIP]
> Auf Aktivierung und Passivierung können wir uns benachrichtigen lassen mithilfe der Annotationen `@PrePassivate` und `@PostActivate`:

```java
@Stateful
public class ShoppingCart implements Serializable {

  // ...

  @PostActivate
  public void restoreState() {
    // ... restore transient fields
  }    

}  
```

### Message Driven Beans (MDB)

- Konsumierung von Nachrichten aus JMS-Queue/Topic
- Aufruf im Hintergrund (asynchron)

```java
@MessageDriven(activationConfig = { 
    // ...
})
public class OrderListener implements MessageListener {

  public void onMessage(Message msg) {
    // ...
  }

}
```

### Timer Beans

Timer Beans sind EJBs, die mit einem EJB-Timer-Service Aufgaben zu einem bestimmten Zeitpunkt oder in einem wiederkehrenden Intervall ausführen können – also quasi „geplante Jobs“ innerhalb des EJB-Containers.

In einem Cluster laufen diese Aufgaben nur einmalig, nicht pro Cluster-Member. Es erfolgt also ein Austausch der Cluster-Member untereinander, auch wenn Instanzen zwischenzeitlich gestoppt und neu gestartet werden.

Wir können diese Aufgaben manuell (programmatisch) starten:

```java
@Stateless
public class CleanupBean {

  @Resource
  TimerService timerService;

  public void scheduleCleanup() {
    timerService.createTimer(60000, "cleanup"); // Startet in 60 Sekunden
  }

  @Timeout
  public void doCleanup(Timer timer) {
    System.out.println("Running cleanup task: " + timer.getInfo());
  }
}
```

Es ist aber auch deklarativ möglich:

```java
@Stateless
public class ReportScheduler {

  @Schedule(
    hour = "2", 
    minute = "0", 
    persistent = false
  )
  public void generateNightlyReport() {
    System.out.println("⏰ Nightly Report is running at 2 AM");
  }
}
```

Dabei bedeutet die `persistent`-Angabe:

- `persistent = true` (Standard)
  - Der Timer wird vom EJB-Container in der Datenbank oder im Timer-Store gespeichert.
    - Timer müssen dann auch mit einer persistence-fähigen Datenquelle (z.B. JTA-DataSource) arbeiten.
    - Der TimerStore kann je nach App Server z.B. eine Tabelle wie `TIMER_TABLE` in der Datenbank anlegen.
  - Nach einem Server-Crash oder -Restart wird der Timer wiederhergestellt und läuft weiter.
- `persistent = false`
  - Der Timer ist nur im Speicher vorhanden (in-memory).
  - Nach einem Neustart des Servers ist der Timer weg und wird nicht reaktiviert.

### Startup-Beans

Startup-Beans sind `@Singleton`-EJBs, die beim Start der Anwendung automatisch instanziiert und initialisiert werden. Wir nutzen sie typischerweise, um Initialisierungslogik oder Preloading von Ressourcen durchzuführen.

> [!NOTE]
> Üblicherweise werden EJBs erst initialisiert, wenn sie das erste Mal benötigt werden. (_Lazy Loading_) Startup-Beans werden sofort initialisiert (_Eager Loading_).

> [!NOTE]
> Startup-Methoden können ebenfalls mit einer Transaktion ausgeführt werden.

```java
@Singleton
@Startup
public class AppInitializer {

  @PostConstruct
  public void init() {
    System.out.println("✅ Startup Bean: Initialisierung läuft!");
    // z.B. Cache vorladen, Konfigurationen einlesen, Timer starten etc.
  }

}
```

> [!CAUTION]
> - Vermeide lange oder blockierende Operationen, da sie den Start des Servers verzögern können.
> - Achte auf Fehlerbehandlung, da Exceptions im Startup-Prozess das Starten der Anwendung verhindern.


## Wofür gibt es Local und Remote Interfaces?

EJBs sollen uns ermöglichen, zwischen lokalen (in derselben JVM) und entfernten (in unterschiedlichen JVMs) Aufrufen zu unterscheiden. Diese Schnittstellen regeln genau das.

| Kriterium   | Local Interface     | Remote Interface                                   |
|-------------|---------------------|----------------------------------------------------|
| Annotation  | `@Local`            | `@Remote`                                          |
| Zweck       | Aufruf in einer JVM | Aufruf von einer JVM zu einer anderen              |
| Bedingungen | keine               | Parameter/Rückgabewerte müssen `Serializable` sein |

> [!TIP]
> Eine EJB kann mehrere Interfaces, sowie Local als auch Remote, gleichzeitig implementieren.

```java
@Local
public interface PaymentServiceLocal {
  void processPayment(Order order);
}

@Remote
public interface PaymentServiceRemote {
  void processPayment(Order order);
}

@Stateless
public class PaymentServiceBean 
    implements PaymentServiceLocal, PaymentServiceRemote {

  @Override
  public void processPayment(Order order) {
    // ...
  }

}
```

Remote Interfaces sind dann in anderen JVMs erreichbar. Ein Beispiel für einen Java-Client wäre:

```java
public class EjbRemoteClient {

  public static void main(String[] args) throws Exception {
    Properties jndiProps = new Properties();
    // Beispiel für WildFly
    jndiProps.put(
      Context.INITIAL_CONTEXT_FACTORY, 
      "org.wildfly.naming.client.WildFlyInitialContextFactory"
    );
    jndiProps.put(
      Context.PROVIDER_URL, 
      "http-remoting://localhost:8080"
    );
    jndiProps.put(
      Context.URL_PKG_PREFIXES,
      "org.jboss.ejb.client.naming"
    );

    try (Context ctx = new InitialContext(jndiProps)) {
      // JNDI-Name hängt vom Server und der App-Struktur ab
      PaymentServiceRemote paymentService
        = (PaymentServiceRemote) ctx.lookup(
            "ejb:/myApp/PaymentServiceRemote!com.example.ejb.PaymentServiceRemote"
          );
      // Aufruf der Remote-EJB
      Order order = new Order();
      // ..
      paymentService.processPayment(order);
    }
  }

}
```
