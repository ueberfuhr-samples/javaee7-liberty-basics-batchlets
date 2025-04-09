# Transaktionen

## Inhalte dieser Seite

1. [Was sind Transaktionen?](#was-sind-transaktionen)
2. [Wie können Transaktionen gesteuert werden?](#wie-können-transaktionen-gesteuert-werden)
3. [Was passiert bei Exceptions?](#was-passiert-bei-exceptions)
4. [Beispiele für Rollbackverhalten in verschachtelten Aufrufen](#beispiele-für-rollbackverhalten-in-verschachtelten-aufrufen)
5. [Was bedeutet "marked for rollback"?](#was-bedeutet-marked-for-rollback)
6. [Was sind Verteilte Transaktionen (XA)?](#was-sind-verteilte-transaktionen-xa)
7. [Was sind Best Practices und Bad Practices?](#was-sind-best-practices-und-bad-practices)

## Was sind Transaktionen?

In der IT, besonders bei Datenbanken, ist eine Transaktion eine logische Einheit aus einer Reihe von Operationen (z. B. mehrere SQL-Befehle), die als Ganzes ausgeführt werden muss. Hier gelten die sogenannten **ACID**-Prinzipien:

- **Atomicity:** Alles oder nichts wird durchgeführt.
- **Consistency**: Die Datenbank bleibt in einem gültigen Zustand.
- **Isolation**: Transaktionen beeinflussen sich nicht gegenseitig.
- **Durability**: Nach Abschluss der Transaktion bleiben die Änderungen bestehen, auch bei einem Systemausfall.

## Wie können Transaktionen gesteuert werden?

Es gibt mehrere Möglichkeiten, zwischen denen jeweils sinnvoll ausgewählt werden sollte:

### Steuerung über JDBC (Java SE)

JDBC selbst bietet die Möglichkeit, Transaktionen manuell zu steuern:

```java
public void doSth() {
  try(Connection con = DriverManager.getConnection(url, user, password)) {
    try {
      con.setAutoCommit(false);
      // ...
      con.commit();
    } catch(Exception e) {
      con.rollback();
    }
  }
}
```

> [!CAUTION]
> Im Java-EE-Umfeld sollten wir auf diese Lösung verzichten, weil wir dort standardisierte Lösungen für weiterführende Problemstellungen (wie verteilte Transaktionen) erhalten. Im Java-SE-Umfeld wären Erweiterungen wie [Atomikos](https://www.baeldung.com/java-atomikos) notwendig.

### Steuerung über Java Transaction API (JTA, Java EE)

Die Transaktionssteuerung über JTA wird verwendet, um Transaktionen programmgesteuert (_Bean Managed Transactions, BMT_) oder deklarativ (_Container Managed Transactions, CMT_) zu verwalten. Diese API bietet eine standardisierte Möglichkeit, Transaktionen zu steuern, insbesondere wenn man mit Ressourcen wie Datenbanken oder Messaging-Systemen arbeitet.

Hier ist eine Übersicht, wie das funktioniert:

#### Kernkomponenten

##### `UserTransaction`
- für BMT
- Zugriff per Resource Injection

```java
@Resource
UserTransaction userTransaction;

public void doSomething() {
  try {
    userTransaction.setTransactionTimeout(60); // seconds
    userTransaction.begin();
    // Logik mit Datenbankoperationen
    userTransaction.commit();
  } catch (Exception e) {
    userTransaction.rollback();
  }
}
```

> [!TIP]
> Diese Variante der BMT empfiehlt sich vor allem in Batchlets, weil dort oft die volle Kontrolle über den Lebenszyklus von Transaktionen notwendig ist. (Zwischencommits, Exception Handling beim Commit)

> [!CAUTION]
> Wir sollten sicherstellen, dass die Transaktion am Ende unserer Logik per Commit oder Rollback geschlossen wird. _Offene Transaktionen_ führen z.B. zu Sperren auf Tabellenzeilen und blockieren anderweitige Ressourcen. Es empfiehlt sich zur Sicherheit das Setzen von Transaktions-Timeouts.

##### `TransactionManager`

- für CMT
- Resource Injection bzw. JNDI-Abfrage nicht standardisiert
- ermöglicht Low-Level-Kontrolle über den Transaktionskontext (Interface `Transaction`)

##### `XAResource`

- für verteilte Transaktionen genutzt
- Teil von JTA zur Unterstützung von 2-Phasen-Commit-Protokollen (2PC)
- nicht nativ vom Container zur Verfügung gestellt, sondern von einer Ressource selbst (z. B. von einem XA-fähigen JDBC-Treiber oder einem JMS-Broker)

```java
import javax.sql.XAConnection;
import javax.sql.XADataSource;
import javax.transaction.xa.XAResource;

@Resource(name = "jdbc/MyXADS")
private XADataSource xaDataSource;

public void customXaLogic() throws Exception {
  XAConnection xaConn = xaDataSource.getXAConnection();
  XAResource xaResource = xaConn.getXAResource();
  // ...
}
```

#### Deklarative Transaktionssteuerung (Container-gesteuert)

In CDI-Beans bzw. Enterprise Java Beans (EJBs) kannst du Transaktionen deklarativ mit Annotationen steuern:

```java
@ApplicationScoped
public class MyService {

  @Transactional(Transactional.TxType.REQUIRED)
  public void process() {
    // Container übernimmt begin / commit / rollback
  }
}
```

Oder für EJBs:

```java
@Stateless
public class MyEJB {

  @TransactionAttribute(TransactionAttributeType.REQUIRED)
  public void process() {
    // Container übernimmt begin / commit / rollback
  }
}
```

Die Annotation `@TransactionAttribute` ist Teil des EJB-Standards und daher besser in das Lebenszyklusmodell von EJBs integriert (z.B. für `@PostConstruct`-Methoden)

> [!TIP]
> Der `TxType` bzw. `TransactionAttributeType` (_Propagation Strategy_) kann angegeben werden, um den Container anzuweisen,
> - eine neue Transaktion zu starten (immer oder falls keine vorhanden)
> - laufende Transaktionen zu beenden (oder Fehler zu werfen)

> [!IMPORTANT]
> Bei CMT verwaltet der Container den Lebenszyklus der Transaktion. Das ist einfacher, weil weniger Code zu schreiben ist. Hier gibt es jedoch Grenzen und Fallstricke zu beachten:
> - Es kann nicht auf Fehler beim Commit reagiert werden.
> - Bewegt man sich innerhalb einer bereits existierenden Transaktion und greift in der eigenen Methode dann auf transaktionsfähige Ressourcen zu, dann werden auch diese im offenen (nicht committeten) Zustand gehalten, solange die Transaktion offen ist. Das sorgt v.a. bei Host-Zugriffen (Connectoren) für nicht unerheblichen Ressourcenverbrauch. In solchen Fällen empfiehlt sich die Verwendung von `REQUIRES_NEW` oder `NOT_SUPPORTED`.

## Was passiert bei Exceptions?

Diese Fragestellung ist vor allem bei CMT wichtig. Hier ist für beide Annotationen (`@Transactional` und `@TransactionAttribute`) folgendes Verhalten festgelegt:

- Bei _Unchecked Exceptions_ (`RuntimeException` und `Error`) erfolgt ein automatisches Rollback.
- Bei _Checked Exceptions_ erfolgt kein Rollback.

Es ist bei `@Transactional` jedoch möglich, eine Exception anzugeben, bei der ein Rollback ausgelöst werden soll. Durch die Berücksichtigung von Vererbungshierarchien lässt sich so auch generell ein Rollback bei _Checked Exceptions_ erzwingen. Und es lassen sich Ausnahmen angeben:

```java
@Transactional(
  rollbackOn = Exception.class,
  dontRollbackOn = {
    SQLException.class,
    CustomException.class
  }
)
public void doSomethingChecked() throws Exception {
  throw new Exception("Erzwinge Rollback trotz Checked Exception, aber nicht für SQL und Custom");
}
```

> [!TIP]
> Analog gibt es für EJBs `SessionContext#setRollbackOnly()` bzw. die Möglichkeit, eigene Exceptions mit `@ApplicationException` zu markieren.

```java
@ApplicationException(rollback = true)
public class MyBusinessException extends Exception {
  // checked, aber verursacht Rollback
}
```

## Beispiele für Rollbackverhalten in verschachtelten Aufrufen

Rufen sich zwei EJBs gegenseitig auf, so kommt es auf deren _Propagation Strategy_ an.
Hier ein Beispiel:

```java
@Stateless
public class OuterService {

  @EJB
  private InnerService innerService;

  @TransactionAttribute(TransactionAttributeType.REQUIRED)
  public void doOuterWork() {
    System.out.println("Outer transaction started.");

    // INNER Methode startet eine neue Transaktion
    innerService.doInnerWork();

    // Outer wirft Exception
    throw new RuntimeException("Outer transaction fails");
  }
}

@Stateless
public class InnerService {

  @TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
  public void doInnerWork() {
    System.out.println("Inner transaction started.");
    // hier würde z. B. ein DB-Insert stattfinden
    // commit passiert unabhängig von Outer!
  }
}
```

### Was passiert hier genau?
- `doOuterWork()` startet eine Transaktion (falls keine vorhanden).
- `doInnerWork()` unterbricht die Outer-Transaktion und startet eine eigene Transaktion.
- Die innere Transaktion committet erfolgreich, auch wenn danach in `doOuterWork()` eine Exception geworfen wird.
- Die äußere Transaktion wird zurückgerollt, die innere bleibt committed!

> [!NOTE]
> **Typische Anwendungen von `REQUIRES_NEW`:**
> - Logging/Protokollierung, unabhängig von der Haupttransaktion
> - Technische Audits oder Events persistieren, auch wenn Business-Logik fehlschlägt
> - Kompensations- oder Fallback-Logik

### Visualisierung

```rust
[ Outer Tx (REQUIRED) ]
     |
     |----> [ Inner Tx (REQUIRES_NEW) ] ---- COMMIT
     |
     |----> Exception ----> Outer Tx ROLLBACK
```

### Was passiert, wenn die Exception in `doInnerWork()` auftritt?

Dann werden beide Transaktionen zurückgesetzt.

```rust
[ Outer Tx (REQUIRED) ]
     |
     |----> [ Inner Tx (REQUIRES_NEW) ] ----> Exception + Rollback
     |
     |----> Exception ----> Outer Tx ROLLBACK
```

Ausnahme: Die Exception wird in `doOuterWork()` abgefangen:

```rust
[ Outer Tx (REQUIRED) ]
     |
     |----> [ Inner Tx (REQUIRES_NEW) ] ----> Exception + Rollback
     |
     |----> catch block ----> Outer Tx läuft weiter (commit möglich)
```

### Was passiert, wenn `doInnerWork()` in der äußeren Transaktion läuft?

Verwendet `doInnerWork()` keine eigene Transaktion, und tritt dann eine Exception auf, wird die Transaktion "marked for rollback".
 
## Was bedeutet "marked for rollback"?

In JTA bedeutet der Zustand "marked for rollback", dass eine aktive Transaktion zwingend zurückgerollt werden muss – sie darf nicht mehr erfolgreich committed werden. Es kommt dann zu einer `RollbackException`.

> [!IMPORTANT]
> Dieser Zustand kann nicht mehr rückgängig gemacht werden. Einzige Lösung, damit die folgende Logik funktioniert, wäre ein manuelles Rollback und der Beginn einer neuen Transaktion.

### Wann geschieht dies?

- **Automatisch:**
  - Wenn innerhalb der Transaktion eine `RuntimeException` oder ein `Error` auftritt. (_Unchecked Exception_)
  - Wenn ein Resource-Manager (z. B. eine `XAResource`) ein Problem meldet (z.B. `XA_RBROLLBACK`).
- **Manuell:**
  - Mit `transaction.setRollbackOnly()` bzw.
  - Bei EJBs mit `sessionContext.setRollbackOnly()`.

Hier ein Beispiel mit einer `UserTransaction` (BMT):

```java
@Resource
UserTransaction tx;

public void example() throws Exception {
  tx.begin();

  // Business-Logik...

  // Entscheidung im Code:
  if (someValidationFails) {
    tx.setRollbackOnly();
  }

  // Abfrage wäre möglich:
  if (tx.getStatus() == Status.STATUS_MARKED_ROLLBACK) {
    System.out.println("⚠️ Marked for rollback!");
  }
    
  // Commit nicht mehr möglich
   try {
     tx.commit();
   } catch (RollbackException e) {
     System.out.println("Commit failed: marked for rollback");
     // hier erfolgt der Rollback durch den Container
   }
}
```

## Was sind Verteilte Transaktionen (XA)?

Eine verteilte Transaktion (engl. "distributed transaction") ist eine Transaktion, die über mehrere unterschiedliche Ressourcen oder Systeme läuft und sicherstellt, dass alle beteiligten Systeme entweder gemeinsam committen oder rollbacken – ganz nach dem ACID-Prinzip. Die Abkürzung _XA_ steht dabei für _eXtended Architecture_.

> [!TIP]
> **Beispiel:**
> Wir haben eine Anwendung, die
> - in eine Datenbank schreibt
> - eine Nachricht in eine JMS-Queue legt
> - eine Buchung in einem SAP-System auslöst

Wie wir erkennen können, müssen nicht nur Datenbanken beteiligt sein. Voraussetzung ist aber, dass die jeweiligen Clients/Konnektoren _XA-kompatibel_ sind. Kafka z.B. unterstützt dies nicht.

### Technische Grundlagen

- Verteilte Transaktionen laufen über den JTA `TransactionManager`.
- XA ist ein [standardisiertes](https://pubs.opengroup.org/onlinepubs/009680699/toc.pdf) 2-Phasen-Commit (2PC) Protokoll.
  - **Prepare-Phase:** `TransactionManager` befragt alle Ressourcen zur Commit-Bereitschaft. Alle Ressourcen antworten entweder mit OK (_vote yes_) oder Problem erkannt (_vote no_).
  - **Commit-Phase:** Haben alle mit OK geantwortet, wird ein Commit an alle geschickt. Andernfalls erhalten alle die Anweisung zum Rollback.

### Wofür wird das benötigt?

- Damit Datenintegrität auch über mehrere Systeme hinweg gewährleistet ist.
- Besonders wichtig in Finanzwesen, Logistik oder Telekommunikation, wo Transaktionen über verschiedene Plattformen laufen müssen.

### Nachteile

- Komplexität höher als bei lokalen Transaktionen.
- XA-Teilnehmer sind langsamer als lokale Ressourcen, da 2PC-Protokoll „teuer“ ist.
- Risiko von "In-Doubt"-Transaktionen bei z. B. Systemabstürzen während der Commit-Phase.

### Lassen sich lokale und verteilte Transaktionen mischen?

Sogenannte "_Mixed-Resource Transactions_" sind möglich, aber mit Einschränkungen.

- Ist mindestens eine XA-fähige Ressource in einer Transaktion beteiligt, wird der JTA-`TransactionManager` in den XA-Modus (2PC) wechseln.
- Lokale Ressourcen (z. B. einfache JDBC-Connections ohne XA) werden dann vom `TransactionManager` trotzdem "mitkoordiniert" – über ein Konzept wie "Last Resource Commit Optimization" (LRCO).
  - Die lokale Ressource (z. B. normale DB-Connection) wird als „last resource“ betrachtet und zuerst committet, bevor der eigentliche 2PC mit den XA-Ressourcen startet.
  - Die lokale Ressource nimmt nicht am Prepare-Phase des XA-Protokolls teil, sondern wird "optimiert" behandelt.
  - Wenn die lokale Ressource erfolgreich committet, aber die XA-Teilnehmer danach scheitern, kann es zu Inkonsistenzen kommen (z. B. DB committed, aber JMS nicht).

## Was sind Best Practices und Bad Practices?

### "Mark for rollback" so früh wie möglich setzen und abfragen

Das vermeidet unnötige Ausführung von Logik. Riskante Zugriffe sollten zuerst stattfinden. Der Status sollte zwischendrin und vor Aufruf von `commit()` abgefragt werden.

### Schichtenarchitekturen

Halte Transaktionsgrenzen möglichst in Service-Schichten, nicht in der Domänenschicht. Vermeide es, Entitäten- oder Utility-Klassen wissen zu lassen, ob sie sich in einer Transaktion befinden.

### Spezifische Exceptions nutzen

Vermeide es, alles mit `RuntimeException` oder `Exception` zu fangen. Definiere klare Business-Exceptions und ggf. Technical-Exceptions, die explizit ein Rollback auslösen sollen.

```java
@ApplicationException(rollback = true)
public class BusinessRuleViolationException extends Exception {}
```

So kannst du präzise steuern, wann der Container "marked for rollback" setzt.

### `REQUIRES_NEW` sparsam einsetzen

Nutze `REQUIRES_NEW` nur für echte Isolationsfälle, z. B. Audit, Logging, technische Protokollierung. In Business-Logik lieber propagieren und sauber rollen oder committen.
Unnötige `REQUIRES_NEW` erzeugen unnötig viele eigenständige Transaktionen und können zu "Lost Updates" (Verlust von Änderungen bei konkurrierenden Transaktionen auf demselben Datensatz) führen.

### Transaktionsstatus abfragen, statt Exception-driven Design

Vermeide Konstrukte wie:

```java
public void doSth() {
  // ...
  try {
    tx.commit();
  } catch (RollbackException e) {
    // zu spät für sauberes Recovery
  }
}
```

Prüfe stattdessen vorher den Status.

### `setRollbackOnly()` vs. Exception

`setRollbackOnly()` ist sinnvoll, wenn du den Fluss kontrolliert fortsetzen möchtest, aber der Commit verhindert werden soll. Ist der Fehler so gravierend, dass die Logik unterbrochen werden muss, wirf eine Exception.

### Transaktionen kurz halten

Langlebige Transaktionen blockieren Ressourcen und erhöhen das Timeout-/Deadlock-Risiko. Daher z.B. Logik ohne Notwendigkeit einer Transaktion außerhalb der Transaktion stattfinden lassen. Z.B. sollten Validierungen vor der Transaktion stattfinden.
