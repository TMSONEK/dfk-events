# ADR 0009 – Rückschreiben der MV in die Anmeldungen

Datum: 18.11.2026 · Status: **Vorschlag, wartet auf Freigabe** (Anforderung Captain)

## Kontext

Die MV gleicht jede Anmeldung mit den Mitgliederdaten ab (E-Mail + Nachname). Das Ergebnis – korrigierte Namen, Mitgliedsnummer, geprüfter Mitgliedsstatus – soll zurück in die Anmeldungen, damit die spätere App es nutzen kann (z. B. „Meine Veranstaltungen“ über die Mitgliedsnummer).

## Optionen

| Option | Beschreibung | Bewertung |
|---|---|---|
| **A** | MySQL-Benutzer der MV erhält `UPDATE` nur auf ausgewählte Spalten von `wp_dfk_registrations` | Am schlanksten; die MV schreibt mit ihrem vorhandenen ODBC-Zugang direkt |
| B | Die MV schreibt in eine Eingangstabelle, ein WP-Cron übernimmt die Änderungen über den Service | Mehr Kontrolle und Protokoll, aber eine Tabelle und ein Job mehr |
| C | REST-Endpunkt | Sauberste Lösung, aber HTTP/JSON in VBA; passt erst zur .NET-MV ab 2027 |

## Vorschlag

**Option A** für V1. Beschreibbar sind nur:

- `first_name`, `last_name` – Namenskorrekturen
- `mv_member_no` – zugeordnete Mitgliedsnummer
- `mv_status` – geprüfter Status (Werte der MV, z. B. Mitglied, Interessent)
- `mv_updated_at` – Zeitpunkt der Rückmeldung

Nicht beschreibbar: E-Mail, Anmeldestatus, Warteliste, Zusatzangaben, Token. Nachrücken und Stornieren bleiben im WP-Backend, weil sie Mails auslösen.

## Folgen

- Drei neue Spalten in `wp_dfk_registrations`; die Selbstauskunft `attendee_type` bleibt unverändert daneben stehen.
- Rückgeschriebene Änderungen lösen keine Mail aus.
- Die Mitgliedsnummer ist ein personenbezogenes Datum und wird mit der Bereinigung nach der Veranstaltung gelöscht (ADR 0005).
- Die MV-Seite muss parametrisierte Abfragen verwenden (siehe Hinweis in [mv-schnittstelle.md](../mv-schnittstelle.md)).
- Mit der .NET-MV kann Option A später durch einen REST-Endpunkt ersetzt werden, ohne das Schema zu ändern.
