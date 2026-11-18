# ADR 0010 – Texte und Einstellungen in einer Tabelle, pflegbar über das CRM

Datum: 18.11.2026 · Status: **Angenommen** (Captain, 18.11.2026)

## Kontext

Die automatischen Mails sollen als reine Text-Templates starten und ändern sich selten. Der Captain möchte Pflegeaufgaben möglichst in das CRM (die MV) verlagern, statt sie im WordPress-Backend zu bauen.

## Entscheidung

Eine kleine Schlüssel-Wert-Tabelle `wp_dfk_settings` enthält **alle** Texte und Einstellungen des Plugins:

| Schlüssel | Inhalt |
|---|---|
| `mail_from_name`, `mail_from_address` | Absender der Mails |
| `mail_booking_confirmed_subject` / `_body` | Mail: Buchung bestätigt |
| `mail_booking_waitlist_subject` / `_body` | Mail: Buchung auf der Warteliste |
| `mail_cancelled_subject` / `_body` | Mail: Storno |
| `mail_changed_subject` / `_body` | Mail: Änderung (inkl. Nachrücken) |
| `contact_email` | zentrale Kontaktadresse (Box „Veranstalter“, Fallback) |
| `media_notice` | Hinweis zu Foto- und Filmaufnahmen |
| `purge_days` | Bereinigungsfrist in Tagen (Standard 30) |

- **Mails als reiner Text** mit Platzhaltern (siehe [mailtexte.md](../mailtexte.md)).
- **Startwerte** werden bei der Installation per Migration eingetragen.
- **Pflege über das CRM:** Der MySQL-Benutzer der MV erhält `SELECT` und `UPDATE` auf die Spalte `value` dieser Tabelle (zusätzlich zu ADR 0009). Das Plugin liest die Werte bei jeder Mail neu; ein Cache ist bei dieser Datenmenge nicht nötig.
- **Keine Einstellungsseite in WordPress in V1.** Eine einfache Bearbeitungsseite im Plugin kann später ergänzt werden, falls gewünscht.

## Folgen

- Eine vierte, sehr kleine Tabelle statt einer Einstellungsseite im WordPress-Backend.
- Das CRM wird zur Pflegeoberfläche für Texte. Weitere Pflegeaufgaben können nach demselben Muster ins CRM wandern (z. B. später Storno, ADR 0009).
- Ungültige Werte (z. B. leere Absenderadresse) fängt das Plugin ab und nutzt dann den Startwert; der Fehler wird ohne personenbezogene Daten protokolliert.
- Die MV-Seite muss parametrisierte Abfragen verwenden (siehe mv-schnittstelle.md).
