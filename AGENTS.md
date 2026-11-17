# AGENTS.md – DFK Events

## 1. Projektauftrag

Entwicklung eines eigenständigen WordPress-Plugins namens **DFK Events** zur Verwaltung von Veranstaltungen, Anmeldungen, Teilnehmern und Wartelisten.

Das Plugin ersetzt langfristig The Events Calendar Pro und Event Tickets Plus auf der Webseite dfk.eu.

Das System soll schlank, performant, sicher, responsiv und langfristig wartbar sein.

Die Entwicklung erfolgt zunächst für die Testumgebung `dev.dfk.eu`.

**Projektziel:** Produktivsetzung bis Ende Dezember 2026, vorbehaltlich erfolgreicher Tests und Datenmigration.

## 2. Rolle des Coding Agents

Du bist Ensign Codex und arbeitest unter folgenden Vorgaben:

- Architekturentscheidungen werden nicht eigenmächtig verändert.
- Vor größeren Implementierungen zunächst einen Umsetzungsplan vorlegen.
- Keine unnötigen Abhängigkeiten oder Frameworks einführen.
- Bestehenden Code vor Änderungen analysieren.
- Änderungen möglichst klein, nachvollziehbar und testbar halten.
- Keine produktiven Daten ohne ausdrückliche Freigabe verändern.
- Keine Änderungen eigenmächtig auf produktive Systeme übertragen.
- Nach jeder Implementierung Änderungen und Testergebnisse zusammenfassen.
- Bei unklaren Anforderungen nachfragen, statt Annahmen als Fakten umzusetzen.

## 3. Technologie-Stack

### Backend
- PHP 8.x
- WordPress Plugin API
- WordPress REST API
- MySQL / MariaDB
- Datenbankzugriff über `$wpdb` und geeignete WordPress-APIs
- Objektorientierte Programmierung
- PSR-4-orientiertes Autoloading und Namespaces

### Frontend
- Serverseitiges Rendering mit PHP
- HTML5 und CSS3
- Vanilla JavaScript (ES6+)
- TypeScript nur bei begründetem Bedarf
- Responsive Design / Mobile First

### Nicht verwenden
- React im WordPress-Frontend
- Vue, Angular oder andere zusätzliche Frontend-Frameworks
- jQuery als neue Abhängigkeit
- Große PHP-Dateien mit vermischten Verantwortlichkeiten
- Geschäftslogik innerhalb von Templates
- Direkte SQL-Abfragen in Views oder Controllern

Die separate DFK-App basiert bereits auf React und TypeScript. Sie ist nicht Bestandteil dieses Repositorys.

## 4. Architektur

Verbindliche Architektur: **MVC mit Service- und Repository-Schicht**.

### Models
Repräsentieren die fachlichen Datenobjekte und ihre Eigenschaften.

### Views
PHP-Templates für die Darstellung. Keine Datenbankabfragen oder Geschäftslogik.

### Controllers
Verarbeiten Anfragen, validieren Eingaben und delegieren an Services.

### Services
Enthalten die eigentliche Geschäftslogik, insbesondere:
- Veranstaltungsregeln
- Buchungen
- Kapazitätsprüfung
- Teilnehmerverwaltung
- Wartelisten
- Benachrichtigungen
- Standort- und Entfernungsfunktionen

### Repositories
Kapseln Datenbankzugriffe und Persistenz.

### REST API
Verwendet dieselben Services wie das WordPress-Frontend.

Keine doppelte Geschäftslogik für Webseite und App.

WordPress-Hooks und REST-Endpunkte dienen als Integrationsschicht.

## 5. Datenmodell

### Veranstaltungen

Veranstaltungen werden als WordPress Custom Post Type `dfk_event` verwaltet.

WordPress übernimmt:
- Titel
- Beschreibung
- Beitragsbild
- Veröffentlichungsstatus
- Permalinks
- SEO-Integration

Strukturierte Veranstaltungsdaten werden in eigenen Tabellen gespeichert.

Beispiele:
- Beginn und Ende
- Anmeldefrist
- Maximale Teilnehmerzahl
- Veranstaltungsort
- Teilnehmerkreis
- Buchungsregeln

### Vorgesehene Tabellen

- `{prefix}dfk_events`
- `{prefix}dfk_event_locations`
- `{prefix}dfk_event_bookings`
- `{prefix}dfk_event_attendees`
- `{prefix}dfk_event_fields`
- `{prefix}dfk_event_field_values`

Für die spätere Check-in-Funktion ist zusätzlich eine Tabelle für Check-in-Daten vorgesehen.

Das endgültige Datenbankschema muss vor der Implementierung spezifiziert und freigegeben werden.

Tabellennamen verwenden den konfigurierten WordPress-Präfix.

## 6. Veranstaltungsorte

Veranstaltungsorte werden zentral und wiederverwendbar verwaltet.

Jeder Veranstaltungsort besitzt:
- Eindeutige ID
- Name
- Straße und Hausnummer
- Postleitzahl
- Ort
- Land
- Breitengrad
- Längengrad

Koordinaten werden gespeichert und nicht bei jedem Seitenaufruf neu ermittelt.

Online-Veranstaltungen benötigen keinen physischen Veranstaltungsort.

Die gespeicherten Koordinaten dienen auch der späteren DFK-App.

## 7. Anmeldung und Teilnehmer

Eine Buchung kann mehrere Teilnehmer enthalten.

Die anmeldende Person und die tatsächlichen Teilnehmer sind unterschiedliche Entitäten.

### Verbindliche Anforderungen

- Ein Mitglied kann mehrere Personen anmelden.
- Teilnehmer können DFK-Mitglieder oder Gäste sein.
- Zulässige Teilnehmerarten sind pro Veranstaltung konfigurierbar.
- Mehrfachanmeldungen können pro Veranstaltung aktiviert oder deaktiviert werden.
- Maximale Teilnehmerzahl je Buchung ist konfigurierbar.
- Jeder Teilnehmer besitzt eine eindeutige ID und einen eigenen Status.
- Einzelne Teilnehmer können storniert werden.
- Kapazitäten werden anhand der Teilnehmer und nicht anhand der Buchungen berechnet.
- Gleichzeitige Buchungen dürfen keine Überbuchungen verursachen.

Buchungsvorgänge müssen transaktionssicher implementiert werden.

## 8. Dynamische Zusatzfelder

Administratoren können pro Veranstaltung zusätzliche Formularfelder definieren.

Unterstützte Feldtypen:
- Text
- Mehrzeiliger Text
- Auswahlfeld
- Checkbox
- Datum

Felder können:
- Pflichtfelder oder freiwillig sein
- Für eine gesamte Buchung gelten
- Für jeden Teilnehmer einzeln gelten

Beispiel: Personalausweisnummer bei bestimmten Veranstaltungen.

Für personenbezogene Daten sind insbesondere folgende Anforderungen zu berücksichtigen:
- Datensparsamkeit
- Zweckbindung
- Berechtigungskontrolle
- Konfigurierbare Löschfristen
- Schutz vor unbeabsichtigter Ausgabe in Logs, APIs oder Exporten

Besonders sensible Zusatzinformationen dürfen nicht ohne ausdrückliche fachliche Notwendigkeit erhoben werden.

## 9. Wartelisten

Eine Wartelistenfunktion ist Bestandteil der ersten Version.

Die Architektur muss Mehrfachanmeldungen und individuelle Teilnehmerstatus berücksichtigen.

Noch festzulegen:
- Automatisches oder manuelles Nachrücken
- Verhalten bei Gruppenbuchungen
- Reservierungsfristen
- Benachrichtigungen

Diese Regeln dürfen nicht ohne vorherige Freigabe fest implementiert werden.

## 10. Veranstaltungsübersicht

Die öffentliche Veranstaltungsübersicht muss vollständig responsiv sein.

Anforderungen:
- Chronologische Sortierung
- Filter nach Zeitraum
- Filter nach Regionalgruppe
- Filter nach Entfernung
- Veranstaltungsstatus
- Bildbasierte Kartenansicht
- Optional kompakte Listenansicht
- Gute mobile Bedienbarkeit

### Standortsuche

Nutzer können einen Standort über PLZ/Ort oder freiwillige Browser-Standortfreigabe festlegen.

Die Entfernung wird zunächst als Luftlinie berechnet.

Unterstützt werden:
- Entfernung in Kilometern
- Umkreisfilter
- Entfernungssortierung

Online-Veranstaltungen sollen unabhängig vom Umkreisfilter angezeigt werden.

## 11. Veranstaltungsdetailseite

Eigene responsive PHP-Templates.

Darstellung von:
- Titel
- Veranstaltungsbild
- Beschreibung
- Datum und Uhrzeit
- Veranstaltungsort
- Veranstalter
- Anmeldeschluss
- Verfügbare Plätze beziehungsweise Wartelistenstatus
- Anmeldeformular
- Anfahrtsinformationen
- Kalenderexport

Die Anmeldung muss auch auf Smartphones komfortabel funktionieren.

Bestehende Veranstaltungs-URLs unter `/event/` sollen bei der Migration möglichst erhalten bleiben.

## 12. WordPress-Integration

Das Plugin muss mit der bestehenden DFK-Umgebung zusammenarbeiten:

- WordPress
- Astra Pro / Astra Child Theme
- Elementor Pro
- Rank Math
- WP-Members

Das Plugin darf für seine Kernfunktionen nicht von Elementor abhängig sein.

Die Darstellung soll über eigene Templates und geeignete WordPress-Integrationspunkte erfolgen.

Bestehende WordPress-Benutzerkonten sollen für authentifizierte Mitglieder verwendet werden.

## 13. REST API

Versionierte API unter:

`/wp-json/dfk-events/v1/`

Die API soll später von der DFK-App verwendet werden.

Geplante Funktionen:
- Veranstaltungen abrufen
- Veranstaltungsdetails abrufen
- Veranstaltungsorte abrufen
- Standortbasierte Suche
- Eigene Buchungen abrufen
- Buchungen erstellen und stornieren
- Teilnehmer verwalten, soweit berechtigt

Öffentliche und geschützte Endpunkte sind strikt zu trennen.

Authentifizierung, Autorisierung und Datenschutz müssen bei jedem Endpunkt berücksichtigt werden.

Die API-Struktur soll langfristig stabil und erweiterbar bleiben.

## 14. Spätere App-Integration

Die DFK-App ist eine eigenständige React-/TypeScript-Anwendung.

Mittelfristig geplant:
- Persönliche Veranstaltungsliste
- Anmeldung über die App
- QR-Code-Tickets
- Check-in
- Anwesenheitsverwaltung
- Gegebenenfalls Offline-Check-in

Diese Funktionen werden in Version 1 architektonisch berücksichtigt, aber nicht vollständig implementiert.

## 15. Sicherheit und Qualität

Verbindliche Entwicklungsregeln:

- WordPress Coding Standards berücksichtigen
- Eingaben validieren und bereinigen
- Ausgaben kontextgerecht escapen
- SQL-Abfragen vorbereiten
- Nonces für geeignete WordPress-Aktionen einsetzen
- Berechtigungen serverseitig prüfen
- REST-Endpunkte mit passenden Permission-Callbacks absichern
- Keine Zugangsdaten im Repository
- Keine produktiven Mitgliederdaten in Tests
- Datenbankmigrationen versionieren
- Fehler nachvollziehbar protokollieren, ohne personenbezogene Daten offenzulegen
- Sicherheitsrelevante und geschäftskritische Logik automatisiert testen

## 16. Entwicklung und Git

Repository: `DFK-Events`

Hauptbranch: `main`

Für größere Funktionen separate Feature-Branches verwenden.

Commits sollen klein und nachvollziehbar bleiben.

Keine eigenmächtigen Pushes, Deployments oder Datenbankänderungen ohne Freigabe.

## 17. Aktueller Projektstatus

**Phase: Architektur und Anforderungsdefinition**

Noch nicht freigegeben:
- Datenbankschema
- Vollständige API-Spezifikation
- Implementierung der Geschäftslogik
- Migration der bisherigen Veranstaltungen und Anmeldungen

### Erster Arbeitsauftrag an Ensign Codex

1. Diese AGENTS.md vollständig lesen.
2. Die Architektur und Anforderungen auf Widersprüche oder offene Punkte prüfen.
3. Eine geeignete Projektverzeichnisstruktur vorschlagen.
4. Offene Architekturentscheidungen dokumentieren.
5. Noch keinen Anwendungscode generieren.
6. Auf die Freigabe des Captains warten.

---

**Leitprinzip:**

Saubere Architektur vor schneller Implementierung.

Wartbarkeit vor unnötiger Komplexität.

Eine gemeinsame Geschäftslogik für WordPress und DFK-App.

Keine eigenmächtigen Architekturentscheidungen.