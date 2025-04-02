# Bean Management & Dependency Injection

In Jakarta EE gibt es mehrere Container, die die Aufgabe haben,
- Objekte mit einem gegebenen Lebenszyklus zu verwalten (_Bean Management_)
- Objekte miteinander zu verdrahten (_Dependency Injection_)
- Dienste entsprechend ihrer Verantwortlichkeit zur Verfügung zu stellen

Hier ein kleiner Überblick bzw. Vergleich:

| Kriterium                               | Web                                                                             | CDI                                                                                                    | EJB                                                                                                                                                 | JPA                                         | JNDI                                                        | Batchlet                                                                      |
|-----------------------------------------|---------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------|-------------------------------------------------------------|-------------------------------------------------------------------------------|
| Zentrale Aufgabe                        | HTTP Requestverarbeitung                                                        | Dependency Injection (ohne weitere technologische Abhängigkeiten)                                      | Geschäftslogik & Transaktionen                                                                                                                      | Datenbankzugriffe und ORM                   | Bereitstellung von Server-Ressourcen                        | Hintergrundjobs                                                               |
| Spezifische Aufgaben/Dienste/Funktionen | HTTP Connection & Session Management, Web Security                              | Interceptors                                                                                           | Clusterorchestrierung (Timer, Singleton), Remote Method Invocation for Java Clients (RMI/IIOP), JMS-Anbindung, Interceptors, Transaktionsverwaltung | Persistenz mit ORM                          | Bereitstellung konfigurierter Ressourcen (lokal und global) | Ausführen von Steps und Chunk-Jobs (asynchron), Scheduling und Wiederaufnahme |
| Wichtigste Java-Technologien            | Servlet API, JSP, JSTL, JSF, JAX-WS, JAX-RS, Websockets                         | CDI, Dependency Injection for Java, Interceptors, Bean Validation, JTA                                 | EJBs, Interceptors, JTA                                                                                                                             | JPA, JTA                                    | JNDI                                                        | Batchlets                                                                     |
| Komponenten                             | Servlets/JSPs, Filter, EventListener, WebSockets, Webservices (SOAP, REST)      | POJOs                                                                                                  | Session/Timer/Message-Driven Beans                                                                                                                  | Entity Manager                              | URLs, Connections (DB, JMS, Mail), Remote EJBs, ...         | Batchlets, Chunk-Jobs mit `ItemReader`, `Processor` und `Writer`              |
| Deklarationen und deren Lifecycles      | `@WebServlet`, `@WebFilter`, `@WebListener`, `@Path`, `@WebService` (Singleton) | `@ApplicationScoped`, `@Singleton` (Singleton), `@SessionScoped` (Session), `@RequestScoped` (Request) | `@Stateless`, `@Stateful`, `@MessageDriven`, `@Singleton` (Singleton)                                                                               | -                                           | (_per Konfiguration am Server_) (Singleton)                 | ?? Singleton                                                                  |
| Injection der Komponenten               | -                                                                               | `@Inject`                                                                                              | `@EJB`                                                                                                                                              | `@PersistenceContext` (für `EntityManager`) | `@Resource`                                                 | -                                                                             |
| Zugriff auf ...                         | CDI, EJB, JPA, JNDI                                                             | CDI, JPA, JNDI                                                                                         | CDI, EJB, JPA, JNDI                                                                                                                                 | -                                           | -                                                           | CDI, JNDI ???                                                                 | 

## Was ist Dependency Injection?

Kaffee bringen lassen, statt selber Kaffee holen 😉

## Welche Herausforderungen gibt es bei DI?

- Abhängigkeit von DI Container - funktioniert nur, wenn beide Objekte (die Abhängigkeit und das abhängige Objekt) im Container verwaltet werden
- Inversion of Control ("Kontrolle abgeben")
- Verlust an Übersicht durch loose Kopplung
- Debugging schwieriger

## Welche Vorteile hat DI?

- Separation of Concerns - Trennung der Verantwortlichkeiten
- weniger Boilerplate Code
- loose Kopplung ermöglicht Austauschbarkeit
  - der Umgebung
  - der Abhängigkeit (Mocking! Testbarkeit!)

## Welche Arten von Injection gibt es?

Wir können zweierlei Unterscheiden (beliebig kreuzbar)

### Nach der Syntax

- **Constructor Injection**: über Konstruktorparameter (empfohlen)
- **Field Injection**: Direktzuweisung an Instanzvariable per `@Inject`
- **Method/Setter Injection**: Übergabe als Methodenparameter

> [!NOTE]
> Beim Mischen der Varianten in einer Klasse findet Dependency Injection in dieser Reihenfolge statt.

### Nach Zuordnung der Abhängigkeit(en)

- **Injection by Type** (Standard): Finden der Instanz(en) des Datentyps im Context (1:1 oder 1:n)
- **Injection by Name**: Finden der Instanz nach deren Name (eindeutig)
- **Injection by Qualifier**: Finden der Instanz nach zusätzlicher Annotation

