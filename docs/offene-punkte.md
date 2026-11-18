# DFK Events – Offene Punkte bis zum fertigen Konzept

Stand: 18.11.2026 · Reihenfolge: Konzept fertigstellen → andere laufende Projekte → Umsetzung DFK Events.

**Kein Anwendungscode, bis das Konzept abgeschlossen und die Umsetzung freigegeben ist** (Captain, 18.11.2026).

## Neu entschieden (18.11.2026)

| Punkt | Entscheidung |
|---|---|
| Testumgebung | Staging-Umgebung bei Mittwald |
| PHP-Mindestversion (E-T1) | PHP 8.1 |
| Geocoding der Orte (E-T6) | Automatisch über Nominatim (OpenStreetMap) beim Speichern, manuell korrigierbar. Übertragen wird nur die Ortsadresse. |

## A. Fachliche Entscheidungen (Captain)

| # | Frage | Empfehlung |
|---|---|---|
| A1 | **ADR 0009** – Rückschreiben der MV (Namen, Mitgliedsnummer, geprüfter Status) | Freigeben wie vorgeschlagen |
| A2 | **Rollen und Rechte** – wer legt Veranstaltungen an, wer sieht Anmeldelisten? Nur Geschäftsstelle oder auch Regionalgruppen-Betreuer für ihre eigenen Termine? | V1: nur Geschäftsstelle/Mitgliederservice. Eigene Rechte („Veranstaltungen verwalten“, „Anmeldungen sehen“, „sensible Felder sehen“), damit Regionalgruppen später ergänzt werden können |
| A3 | **Stornierung** – bis wann darf die anmeldende Person selbst stornieren? | Über den Verwaltungslink bis Veranstaltungsbeginn, danach nur der Mitgliederservice |
| A4 | **Mehrfachanmeldung** – wann gilt eine Anmeldung als doppelt? | Gleiche E-Mail der anmeldenden Person bei derselben Veranstaltung; pro Veranstaltung abschaltbar |
| A5 | **Gleiche E-Mail bei mehreren Teilnehmern einer Buchung** (Ehepaar) | Erlaubt |
| A6 | **Online-Zugangslink** – wo sichtbar? | Nur in der Bestätigungsmail und über den Verwaltungslink, nie öffentlich |
| A7 | **Wiederkehrende Veranstaltungen** | Keine Serien; Funktion „Veranstaltung duplizieren“ |
| A8 | **Übersicht** – Kachelansicht mit Bild als Standard, kompakte Liste umschaltbar? | Ja |
| A9 | **Standortsuche der Besucher** – PLZ-Eingabe auflösen über lokale PLZ-Tabelle oder über Nominatim? | Lokale PLZ-Tabelle (rund 8.000 Einträge, einmal importiert): keine Weitergabe der Besuchereingabe, keine Abhängigkeit |
| A10 | **Mailtexte und Absender** | Absender und Texte als Einstellung im Plugin; Entwurf der Texte folgt |
| A11 | **Alte TEC-Adressen** `/event/<alt>/` nach Abschaltung | Weiterleitung auf die Übersicht |
| A12 | **Deinstallation** des Plugins | Daten bleiben erhalten, Löschen nur per ausdrücklicher Einstellung |

## B. Technische Punkte (Empfehlung gilt, sofern kein Einspruch)

| # | Punkt | Empfehlung |
|---|---|---|
| B1 | Autoloading | Eigener kleiner PSR-4-Autoloader; Composer nur für Entwicklungswerkzeuge |
| B2 | Buchen ohne REST | Normaler Formularversand über `admin-post.php`, funktioniert auch ohne JavaScript |
| B3 | Seiten-Cache auf dfk.eu | Klären, welches Caching aktiv ist; Anmeldeformular und freie Plätze vom Cache ausnehmen bzw. per `admin-ajax.php` nachladen |
| B4 | Rank Math | Plugin liefert Titel/Beschreibung; Event-Schema nur einmal ausgeben |
| B5 | Elementor | Für `dfk_event` greifen keine Elementor-Theme-Builder-Vorlagen |
| B6 | Admin-Oberfläche | Klassische Metaboxen, kein React |
| B7 | Tests | PHPUnit (Unit-Tests ohne WordPress für Buchungsregeln), PHPCS mit WordPress Coding Standards |
| B8 | Logging | Schlanker Logger auf Basis `error_log`, ohne personenbezogene Daten |
| B9 | Paralleler Betrieb mit TEC auf Staging | Während der Entwicklung Slug `/dfk-event/`, Umstellung auf `/event/` beim Go-live |
| B10 | Deployment auf Staging | Wie bei der DFK-App per GitHub Actions bei Push auf einen festgelegten Branch; kein eigenständiges Deployment durch den Coding Agent |

## C. Für die MV-Seite (separates Projekt)

- `T_Veranstalter` (mit `InterneID`), `T_Veranstaltungen`, `T_Veranstaltungsorte`, `T_Veranstaltungen_Anmeldungen` anlegen
- Abgleich auf die neuen Views umstellen, **parametrisierte Abfragen** (SQL-Injection-Risiko im heutigen Import)
- Abgleich innerhalb von 30 Tagen nach jeder Veranstaltung sicherstellen
- Optional: Rückschreiben (ADR 0009)
