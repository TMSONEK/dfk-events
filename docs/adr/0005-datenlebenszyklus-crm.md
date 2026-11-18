# ADR 0005 – Datenlebenszyklus: Übernahme in die MV, Löschung nach 30 Tagen

Datum: 17.11.2026, geändert 18.11.2026 · Status: **Angenommen** (Captain)

## Kontext

Die Mitgliederverwaltung (MV) übernimmt die Anmelde- und Teilnehmerdaten aus DFK Events. DFK Events ist damit **kein dauerhafter Datenspeicher** für personenbezogene Daten, sondern ein Zwischenspeicher bis zur Übernahme in die MV.

## Entscheidung

1. Die MV übernimmt die Daten, normalerweise direkt nach der Veranstaltung.
2. **30 Tage nach dem Veranstaltungstermin** (maßgeblich ist das Ende der Veranstaltung, **nicht** das Anmeldedatum) löscht DFK Events automatisch alle Anmeldungen der Veranstaltung (Entscheidung 18.11.2026). Die Frist ist als Einstellung änderbar.
3. Es gibt **keine Übernahmebestätigung** durch die MV. Die feste Frist ist die einzige Regel.
4. Für sensible Zusatzfelder gilt dieselbe Regel.
5. Die MV erstellt **alle Auswertungen** (z. B. nach Regionalgruppe, Mitglied/Nichtmitglied). DFK Events braucht keine eigenen Statistik- oder Auswertungsfunktionen.

## Umsetzung

- Ein täglicher WP-Cron löscht die Anmeldungen aller Veranstaltungen, deren Ende mehr als 30 Tage zurückliegt, und setzt `purged_at` an der Veranstaltung.
- Die Veranstaltung selbst (Beitrag, Termin, Ort) bleibt erhalten.

## Folgen

- Die MV muss innerhalb von 30 Tagen nach einer Veranstaltung mindestens einmal abgleichen, sonst gehen Anmeldungen verloren. Bei manuellem Abgleich über die MV ist das organisatorisch sicherzustellen.
- Die Tabelle `wp_dfk_mv_sync` und das zugehörige Schreibrecht der MV entfallen.
- Nach der Bereinigung fehlen die Anmeldungen in der View. Die MV darf daraus nicht auf ein Storno schließen (Stornos sind bis dahin mit Status `storniert` sichtbar).
- Verwaltungslinks werden mit der Bereinigung ungültig.
