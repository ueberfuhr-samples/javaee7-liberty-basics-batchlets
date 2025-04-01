# Java EE Grundlagen

## Was ist Java EE?
- Erweiterung der Standard Edition (SE) um
  - zusätzliche APIs für Anforderungen wie verteilte Architekturen, Persistenz, Skalierbarkeit...
  - eine Laufzeitumgebung, die für diese Anforderungen Dienste anbietet (Application Server)

## Was ist J2EE und Jakarta EE?
- J2EE bezieht sich auf alle JavaEE-Versionen von 1.2 bis 1.4 ("Java 2"). Ab dem Nachfolger (Java 5) wurde "Java EE" üblich.
- Jakarta EE ist der Nachfolger vom Java EE. Mit Version 8 gab Oracle die Entwicklung an die Eclipse Foundation. Aus Java EE 8 entstand Jakarta EE 8, seitdem erfolgt dort die Weiterentwicklung.

## Servlets

### Was sind Servlets?

Servlets sind Objekte, die HTTP-Requests verarbeiten und HTTP-Responses generieren.

### Was sind JSPs?

Java Server Pages (JSPs) sind Servlets, die nicht in Java-Code geschrieben werden, sondern als HTML-Seite mit "dynamischen Elementen".
Oft werden diese mit Servlet-Klassen kombiniert, sodass die Logik in Java-Code, die Ausgabegenerierung in einer JSP implementiert werden kann.

### Welche Web Scopes gibt es?

- **Page**: lokale Variable
- **Request**: für eine Anfrageverarbeitung
- **Session**: für mehrere Anfragen desselben Clients (Vorsicht: Speicherverbrauch und Serialisierbarkeit sind Stolperfallen)
- **Application**: eine Instanz für alle Clients über alle Requests hinweg

> [!NOTE]
> Servlets selbst sind Application-Scoped, d.h. es gibt nur eine Instanz eines Servlets über die gesamte Laufzeit der Anwendung hinweg.

## CDI (Contexts and Dependency Injection)

### Was ist Dependency Injection?

Kaffee bringen lassen, statt selber Kaffee holen 😉

### Welche Herausforderungen gibt es bei DI?

- Abhängigkeit von DI Container - funktioniert nur, wenn beide Objekte (die Abhängigkeit und das abhängige Objekt) im Container verwaltet werden
- Inversion of Control ("Kontrolle abgeben")
- Verlust an Übersicht durch loose Kopplung
- Debugging schwieriger

### Welche Vorteile hat DI?

- Separation of Concerns - Trennung der Verantwortlichkeiten
- weniger Boilerplate Code
- loose Kopplung ermöglicht Austauschbarkeit
  - der Umgebung
  - der Abhängigkeit (Mocking! Testbarkeit!)

### Welche Arten von Injection gibt es?

Wir können zweierlei Unterscheiden (beliebig kreuzbar)

#### Nach der Syntax

- **Constructor Injection**: über Konstruktorparameter (empfohlen)
- **Field Injection**: Direktzuweisung an Instanzvariable per `@Inject`
- **Method/Setter Injection**: Übergabe als Methodenparameter

> [!NOTE]
> Beim Mischen der Varianten in einer Klasse findet Dependency Injection in dieser Reihenfolge statt.

#### Nach Zuordnung der Abhängigkeit

- **Injection by Type** (Standard): Finden der Instanz(en) des Datentypes im Context (1:1 oder 1:n)
- **Injection by Name**: Finden der Instanz nach deren Name (eindeutig)
- **Injection by Qualifier**: Finden der Instanz nach zusätzlicher Annotation

### Welche CDI-Features gibt es noch?

- **Producer Methods und Producer Fields**: Factory Pattern für Dependency Injection
- **CDI Events**: Loose Kopplung, Observer Pattern (1:n), synchron und asynchron (innerhalb des CDI Containers)
- **Interceptors**: Auslagern von technischem Code aus den Methoden heraus in ein Proxy-Objekt, Einbinden über Annotation

### Was ist _Dependency Injection for Java_?

_DI for Java_ ist eine Standard-API für DI. Sie enthält keine Implementierungen und wird von Frameworks wie CDI, Spring, Google Guava ... aufgegriffen.

Folgende API-Bestandteile sind in diesem Standard abgedeckt:
- Annotation `Inject`
- Lazy Injection per `Provider` interface
- Injection by Name / by Qualifier
- Scope (abstrakt)

NICHT abgedeckt (also CDI-spezifisch) sind z.B.:
- Web Scopes
- Injection aller Implementierungen eines Interfaces (`javax.enterprise.inject.Instance`)
