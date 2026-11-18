# DFK Events – Offene Punkte bis zum fertigen Konzept

Stand: 18.11.2026 · Reihenfolge: Konzept fertigstellen → andere laufende Projekte → Umsetzung DFK Events.

**Kein Anwendungscode, bis das Konzept abgeschlossen und die Umsetzung freigegeben ist** (Captain, 18.11.2026).

## Entschieden (18.11.2026)

| # | Thema | Entscheidung |
|---|---|---|
| – | Testumgebung | Staging-Umgebung bei Mittwald |
| E-T1 | PHP | Mindestversion PHP 8.1 |
| E-T6 | Geocoding der Orte | Automatisch über Nominatim beim Speichern, manuell korrigierbar |
| A1 | Rückschreiben der MV | **ADR 0009 angenommen** |
| A2 | Rollen und Rechte | **Nur WordPress-Administratoren** legen Veranstaltungen an und sehen Anmeldungen. Intern trotzdem eigene Rechte (Capabilities), die nur der Administrator-Rolle zugewiesen werden – so lässt sich das später ohne Umbau erweitern |
| A3 | Stornierung | **Keine Selbst-Stornierung.** Teilnehmer melden sich beim Mitgliederservice (Mail/Telefon), der storniert. Kann sich später ändern. Folge: Der Verwaltungslink per Token entfällt in V1 |
| A4 | Mehrfachanmeldung | Doppelt ist eine **zweite Buchung mit identischer E-Mail der anmeldenden Person** bei derselben Veranstaltung. Innerhalb einer Buchung ist dieselbe E-Mail erlaubt. Pro Veranstaltung abschaltbar (AGENTS.md §7) |
| A5 | Gleiche E-Mail in einer Buchung | Erlaubt |
| A7 | Wiederkehrende Veranstaltungen / Duplizieren | **Nicht in V1**, „nice to have“ für später |
| A8 | Übersicht | Kacheln mit Bild als Standard, kompakte Liste umschaltbar; **bisheriges Erscheinungsbild möglichst beibehalten; Gestaltung mit Elementor muss möglich sein** (siehe B5) |
| A9 | Standortsuche der Besucher | **Nominatim.** Die Anfrage stellt der Server, nicht der Browser – die IP des Besuchers geht nicht an Nominatim. Ergebnisse je PLZ/Ort werden zwischengespeichert (Nutzungsrichtlinie von Nominatim). Die lokale PLZ-Tabelle entfällt |
| A10 | Mailtexte und Absender | Als Einstellung im Plugin; Entwurf der Texte folgt |
| A11 | Alte TEC-Adressen | **Keine Logik im Plugin.** Weiterleitungen werden per Redirect (z. B. Redirect-Manager von Rank Math) eingerichtet |
| A12 | Deinstallation | Daten bleiben erhalten, Löschen nur per ausdrücklicher Einstellung |
| B1–B4, B6–B10 | Technik | Empfehlungen übernommen (siehe unten) |
| B5 | Elementor | **Geändert wegen A8:** Elementor wird unterstützt, nicht ausgeschlossen (siehe unten) |

### Technische Punkte (übernommen)

| # | Punkt | Festlegung |
|---|---|---|
| B1 | Autoloading | Eigener kleiner PSR-4-Autoloader; Composer nur für Entwicklungswerkzeuge |
| B2 | Buchen ohne REST | Normaler Formularversand über `admin-post.php`, funktioniert auch ohne JavaScript |
| B3 | Seiten-Cache | Anmeldeformular und freie Plätze vom Cache ausnehmen bzw. per `admin-ajax.php` nachladen – **welches Caching aktiv ist, noch klären** |
| B4 | Rank Math | Plugin liefert Titel/Beschreibung; Event-Schema nur einmal ausgeben |
| B5 | Elementor | Kernfunktionen bleiben unabhängig von Elementor (AGENTS.md §12): Das Plugin bringt eigene Templates mit. **In Schritt 2** kommen **Shortcodes** dazu (z. B. `[dfk_events]` für die Übersicht, `[dfk_event_booking]` für das Anmeldeformular, `[dfk_event_details]`), die in Elementor-Seiten und Theme-Builder-Vorlagen eingesetzt werden können. Eine Elementor-Vorlage für `dfk_event` hat Vorrang vor dem Plugin-Template |
| B6 | Admin-Oberfläche | Klassische Metaboxen, kein React |
| B7 | Tests | PHPUnit (Unit-Tests ohne WordPress für Buchungsregeln), PHPCS mit WordPress Coding Standards |
| B8 | Logging | Schlanker Logger auf Basis `error_log`, ohne personenbezogene Daten |
| B9 | Paralleler Betrieb mit TEC auf Staging | Während der Entwicklung Slug `/dfk-event/`, Umstellung auf `/event/` beim Go-live |
| B10 | Deployment auf Staging | Per GitHub Actions bei Push auf einen festgelegten Branch (wie DFK-App); kein eigenständiges Deployment durch den Coding Agent |

### Nachträge (18.11.2026)

| # | Thema | Entscheidung |
|---|---|---|
| O1 | Wo storniert der Mitgliederservice? | **V1: in der Anmeldeliste im WordPress-Backend** (Platz wird frei, Stornomail geht raus). **Später zusätzlich aus der MV/dem CRM** – dann wird ADR 0009 um den Anmeldestatus erweitert |
| O2 | Online-Zugang (z. B. Zoom-Link) | **Nicht im Plugin.** Der Zugang wird manuell per Mail verschickt; eine Zoom-Anbindung ist für später angedacht, aber offen. Das Feld `online_url` entfällt |
| O3 | Bisheriges Erscheinungsbild | Screenshots ausgewertet: [erscheinungsbild.md](erscheinungsbild.md) |
| O4 | Seiten-Caching | Derzeit nur minimal aktiv, wird vom Captain überarbeitet. Das Plugin geht davon aus, dass Seiten gecacht werden können (Formular und freie Plätze werden dynamisch geladen, B3) |
| B5 | Elementor | **Präzisiert:** V1 mit eigenen Templates; **Shortcodes in Schritt 2**. Zur Not wird HTML in einem Elementor-Container eingebunden |

## Noch offen

| # | Frage | Empfehlung |
|---|---|---|
| O5 | Google-Karte auf der Detailseite (lädt ohne Einwilligung Inhalte von Google) | Link „In Google Maps öffnen“ statt Einbettung, oder Karte erst nach Klick |
| O6 | Foto-/Film-Hinweis, heute in jeder Beschreibung kopiert | Einmal als Einstellung, automatisch unter jeder Veranstaltung und im Formular; pro Veranstaltung abschaltbar |
| O7 | E-Mail in der Box „Veranstalter“ | Optionale E-Mail je Veranstalter, sonst zentrale Adresse aus den Einstellungen |

## C. Für die MV-Seite (separates Projekt)

- `T_Veranstalter` (mit `InterneID`), `T_Veranstaltungen`, `T_Veranstaltungsorte`, `T_Veranstaltungen_Anmeldungen` anlegen
- Abgleich auf die neuen Views umstellen, **parametrisierte Abfragen** (SQL-Injection-Risiko im heutigen Import)
- Abgleich innerhalb von 30 Tagen nach jeder Veranstaltung sicherstellen
- Rückschreiben von Namen, Mitgliedsnummer und geprüftem Status (ADR 0009)
