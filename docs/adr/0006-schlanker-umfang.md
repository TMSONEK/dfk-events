# ADR 0006 – Schlanker Umfang: keine Migration, nur Deutschland

Datum: 18.11.2026 · Status: **Angenommen** (Captain)

## Kontext

Die ersten Entwürfe enthielten Vorkehrungen für die Migration der TEC-Daten, für Zeitzonen und für große Datenmengen.

## Entscheidung

- **Keine Migration** der bestehenden Veranstaltungen aus The Events Calendar. Anmeldungen werden bei Bedarf von Hand übertragen.
- **Keine internationalen Veranstaltungen.** Alle Veranstaltungen finden in Deutschland statt oder online.
- **Größenordnung:** 100–120 Veranstaltungen pro Jahr. Das System wird dafür ausgelegt und nicht aufgebläht.
- **Nur interne Nutzung durch den DFK**, kein kommerzielles oder öffentlich verteiltes Plugin (Nachtrag 18.11.2026). Daraus folgt:
  - Nur Deutsch; keine Übersetzungsdateien, Texte direkt auf Deutsch im Code bzw. in den Einstellungen.
  - DFK-Besonderheiten (Farben, Texte, Veranstalter-Typen, Anbindung an die MV) dürfen fest eingebaut sein; keine allgemeine Konfigurierbarkeit „für alle Fälle“.
  - Einstellungsseite nur für das, was der DFK wirklich ändert (Absender, Mailtexte, Foto-Hinweis, zentrale E-Mail, Bereinigungsfrist).
  - Kein Template-Override über das Child-Theme; Anpassungen erfolgen direkt im Plugin.
  - Keine Lizenz-, Update- oder Marketplace-Mechanik; Verteilung nur über das eigene Repository.
  - Unverändert gelten: Sicherheit, Datenschutz, WordPress Coding Standards, Tests.

## Folgen

- Keine Migrationsspalten, kein Migrationswerkzeug, keine Zuordnung alter Veranstalter-Schreibweisen im Plugin.
- Zeiten werden nur in deutscher Ortszeit gespeichert.
- Keine Optimierungen für große Datenmengen (Zähler, Geo-Indizes, Bounding-Box-Suche).
- Neue Teilnehmer-IDs müssen trotzdem oberhalb der alten TEC-IDs beginnen, weil die MV die alten Anmeldungen behält (Schema S4).
- Alte Veranstaltungsseiten unter `/event/` entfallen, wenn TEC deaktiviert wird. Neue Veranstaltungen nutzen wieder `/event/<slug>/`. Ob alte Adressen auf die Übersicht weitergeleitet werden, ist noch zu entscheiden.
