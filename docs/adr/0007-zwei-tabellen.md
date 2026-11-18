# ADR 0007 – Schlanke Tabellenstruktur: Veranstaltungen, Orte, Anmeldungen

Datum: 18.11.2026 · Status: **Angenommen** (Captain)

## Kontext

AGENTS.md §5 sah ursprünglich sechs Tabellen vor (Events, Orte, Buchungen, Teilnehmer, Felder, Feldwerte). Für 100–120 Veranstaltungen pro Jahr ist das zu aufwendig. Auch die bisherige Struktur in der MV gilt als zu verschachtelt.

## Entscheidung

Drei Tabellen im Plugin:

- `wp_dfk_events` – eine Zeile je Veranstaltung, inklusive Definition der Zusatzfelder (JSON).
- `wp_dfk_venues` – Veranstaltungsorte mit Geokoordinaten, wiederverwendbar. Die Koordinaten werden für die Umkreissuche, die App und das Python-Landkartenskript gebraucht.
- `wp_dfk_registrations` – eine Zeile je Person. Eine **Buchungs-ID** fasst die Personen einer Buchung zusammen. **Warteliste als Flag**, keine eigene Tabelle.

Die MV wird gleich aufgebaut: `T_Veranstaltungen`, `T_Veranstaltungsorte`, `T_Veranstaltungen_Anmeldungen` (mit Buchungs-ID und Wartelisten-Flag), dazu `T_Veranstalter`.

## Folgen

- Keine eigenen Tabellen für Buchungen, Zusatzfelder und Feldwerte; Zusatzfelder und ihre Werte liegen als JSON vor, sensible Werte darin verschlüsselt.
- Koordinaten werden in WordPress beim Speichern eines Orts ermittelt und über die View an die MV geliefert. Die MV muss nicht selbst geocodieren.
- Die Views für die MV bilden die MV-Tabellen nahezu 1:1 ab.
