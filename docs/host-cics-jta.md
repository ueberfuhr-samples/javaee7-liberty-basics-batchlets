# Host-/CICS-Anfragen mit JTA

Bei HOST- und CICS-Anfragen (_Customer Information Control System_, also IBM's Transaktionsmonitor) in Verbindung mit einer Transaktion per JTA (Java Transaction API) sind einige technische und organisatorische Aspekte zu berücksichtigen. Hier sind die wichtigsten Punkte:

## 🔁 Transaktionskontext und XA-Ressourcen
JTA verwendet in der Regel XA-Transaktionen, um mehrere Ressourcen (z.B. Datenbank, Messaging, CICS) in einer globalen Transaktion zu koordinieren.

Der CICS-Adapter oder -Connector muss XA-fähig sein, um sich korrekt in den JTA-Transaktionskontext einzubinden.

> [!CAUTION]
> Nicht jeder CICS-Zugriff unterstützt XA! Manche Legacy-Systeme arbeiten nur mit lokalen Transaktionen oder benötigen spezielle Bridges.

## 🧩 Verbindung über CICS Transaction Gateway (CTG)
Häufig erfolgt die Verbindung über den CICS Transaction Gateway, z.B. über:
- JCA (Java Connector Architecture) mit einem XA-fähigen Resource Adapter
- SOAP- / REST- / TCP-basierte Schnittstellen

Der JCA-Adapter muss korrekt in den Application Server eingebunden sein, um an JTA teilzunehmen.

## ⏳ Zeitmanagement und Timeout-Konfiguration
JTA-Transaktionen unterliegen einem zentralen Timeout. HOST-Systeme (wie CICS) können jedoch längere Antwortzeiten haben. Transaction Timeouts müssen daher aufeinander abgestimmt sein:
- JTA-Timeout
- CTG-Timeout
- CICS-Timeout
- Netzwerk-Timeouts

## 🔐 Sicherheit / Authentifizierung
HOST-Systeme verlangen meist spezifische Authentifizierung (RACF, Kerberos, etc.). Die Sicherheitskontexte müssen innerhalb der JTA-Transaktion konsistent bleiben, vor allem bei mehreren Hops.

## 🧨 Fehler- und Rollbackverhalten
Bei einem Fehler in einer Ressource (z.B. DB oder CICS) wird die gesamte JTA-Transaktion zurückgerollt.

> [!CAUTION]
> CICS muss Rollback-fähig sein – d.h. der CICS-Request muss als Teil der XA-Transaktion aufgerufen worden sein. Falls CICS nicht XA-fähig ist, könnte eine inkonsistente Datenlage entstehen.

## 📊 Logging & Tracing
Für Troubleshooting sind Transaction Logs und Traces aus allen beteiligten Komponenten notwendig:
- Applikationsserver (JTA-Logs)
- CTG Logs
- CICS Logs
