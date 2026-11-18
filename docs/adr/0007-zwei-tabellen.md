# ADR 0007 – Zwei Tabellen: Veranstaltungen und Anmeldungen

Datum: 18.11.2026 · Status: **Angenommen** (Captain)

## Kontext

AGENTS.md §5 sah sechs Tabellen vor (Events, Orte, Buchungen, Teilnehmer, Felder, Feldwerte). Für 100–120 Veranstaltungen pro Jahr ist das zu aufwendig. Auch die bisherige Struktur in der MV gilt als zu verschachtelt.

## Entscheidung

- **Zwei Tabellen** im Plugin:
  - `wp_dfk_events` – eine Zeile je Veranstaltung, inklusive Ort und Definition der Zusatzfelder (JSON).
  - `wp_dfk_registrations` – eine Zeile je Person. Eine **Buchungs-ID** fasst die Personen einer Buchung zusammen.
- **Warteliste als Flag** in der Anmeldung, keine eigene Tabelle.
- Die MV wird gleich aufgebaut: `T_Veranstaltungen`, `T_Veranstaltungen_Anmeldungen` (mit Buchungs-ID und Wartelisten-Flag), dazu `T_Veranstalter`.

## Folgen

- Orte sind nicht mehr zentral wiederverwendbar (Abweichung von AGENTS.md §6). Beim Anlegen kann ein Ort aus einer früheren Veranstaltung übernommen werden.
- Zusatzfelder und ihre Werte liegen als JSON vor; sensible Werte darin verschlüsselt.
- Die Views für die MV bilden die MV-Tabellen nahezu 1:1 ab.
- AGENTS.md §5, §6 und §9 werden entsprechend angepasst.
