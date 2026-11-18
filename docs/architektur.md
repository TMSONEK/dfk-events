# DFK Events – Architekturvorschlag und offene Entscheidungen

Stand: 18.11.2026 · Status: **Entwurf, teilweise entschieden – Rest wartet auf Freigabe durch den Captain**

> **Hinweis 18.11.2026:** Mit ADR 0006–0008 wurde der Umfang deutlich verschlankt (drei Tabellen, keine Migration, Warteliste manuell). Abschnitt 0 und 2 sind aktuell. Die Tabellen in Abschnitt 4 und 5 sind die ursprüngliche Analyse; Punkte, die durch die Verschlankung entfallen, sind dort nicht mehr einzeln nachgezogen.

## 0. Bereits getroffene Entscheidungen

| ID | Entscheidung | ADR |
|---|---|---|
| E-F1 | Keine Bezahlung. Alle Veranstaltungen sind kostenfrei, das Plugin ist ein reines **RSVP-System**. | [0001](adr/0001-keine-bezahlung.md) |
| E-F3 | **Buchen ist immer ohne Login möglich.** Ein Login füllt nur Daten vor und ist keine Voraussetzung. | [0002](adr/0002-buchung-ohne-login.md) |
| E-F4 | Teilnehmerart per **Dropdown „Mitglied / Nichtmitglied“** für jede Person der Buchung, auch für mit angemeldete Gäste. Selbstauskunft ohne Prüfung, keine Mitgliedsnummer. „Gast“ = Begleitperson, nicht Nichtmitglied. | [0003](adr/0003-teilnehmerart-selbstauskunft.md) |
| – | Buchung = anmeldende Person + 1…n Teilnehmer. Die anmeldende Person nimmt standardmäßig selbst teil (Häkchen), kann aber auch nur andere anmelden. | [0004](adr/0004-anmeldende-person-und-teilnehmer.md) |
| E-F21 | Pflichtfelder je Teilnehmer: Vorname, Nachname, Mitglied/Nichtmitglied, E-Mail. Weitere Felder optional pro Event. | [0004](adr/0004-anmeldende-person-und-teilnehmer.md) |
| E-F23 | Bestätigungsmail vorerst nur an die anmeldende Person, mit Liste aller angemeldeten Personen. | – |
| E-F24 | Zusatzfelder können als „sensibel“ markiert werden (eingeschränkte Sichtbarkeit, verschlüsselt, nicht in API/Mails/Logs, automatische Löschung). | – |
| E-F15 | Alle Anmeldungen einer Veranstaltung werden **30 Tage nach dem Veranstaltungstermin** (Ende der Veranstaltung, nicht Anmeldedatum) automatisch gelöscht. Die MV gleicht vorher ab. | [0005](adr/0005-datenlebenszyklus-crm.md) |
| E-F26/27 | Schnittstelle zur MV: versionierte DB-Views mit eingeschränktem MySQL-Benutzer; keine Sync-Bestätigung, Bereinigung nach fester Frist. | [mv-schnittstelle.md](mv-schnittstelle.md) |
| E-F5 | Jedes Event gehört genau einem Veranstalter: DFK, Regionalgruppe, Netzwerk (oder Kooperation). Taxonomie `dfk_unit`, Kennung `T_Veranstalter.InterneID` der MV, einmaliger CSV-Import + manuelle Pflege. Reine Online-Events gehören immer dem DFK, hybride dem Veranstalter vor Ort. | [datenbankschema.md](datenbankschema.md) |
| E-F6 | Eventstatus: findet statt / abgesagt / verschoben. | – |
| E-F7 | Kein Freitext mehr: Veranstalter = Auswahl aus der Liste `T_Veranstalter` der MV, Kennung `InterneID`. Genau einer je Event (Pflicht). | [datenbankschema.md](datenbankschema.md) |
| E-F16 | Keine Migration der TEC-Veranstaltungen; Anmeldungen bei Bedarf von Hand. Nur Deutschland. 100–120 Veranstaltungen pro Jahr – schlank halten. | [0006](adr/0006-schlanker-umfang.md) |
| – | Drei Tabellen: Veranstaltungen (Zusatzfelder als JSON), Orte (mit Geokoordinaten) und Anmeldungen (eine Zeile je Person, Buchungs-ID, Wartelisten-Flag). MV wird gleich aufgebaut. | [0007](adr/0007-zwei-tabellen.md) |
| E-F9 | Warteliste manuell: kein automatisches Nachrücken, Mitgliederservice entscheidet. Passt eine Gruppe nicht vollständig, kommt die ganze Buchung auf die Warteliste. Mails nur bei Buchung, Storno, Änderung. | [0008](adr/0008-warteliste-und-mails.md) |
| – | **Vorschlag:** MV schreibt Namenskorrekturen, Mitgliedsnummer und geprüften Status direkt in die Anmeldungen zurück (spaltengenaue MySQL-Rechte). | [0009](adr/0009-rueckschreiben-mv.md) |
| – | Die App bucht über die WordPress-Seite (reduzierte Anmeldeseite, Prüfung am Prototyp), keine Schreib-API. Termine liest das App-Backend über **einen** öffentlichen, nur lesenden und abgesicherten Endpunkt `GET /wp-json/dfk-events/v1/events`. | [app-anbindung.md](app-anbindung.md) |
| E-F17 | Kein Double-Opt-in in V1; Spamschutz über Honeypot und Rate-Limit. | [datenbankschema.md](datenbankschema.md) |
| E-T1/E-T6 | PHP 8.1; Geocoding der Orte automatisch über Nominatim, manuell korrigierbar; Testumgebung: Staging bei Mittwald. Offene Restpunkte: [offene-punkte.md](offene-punkte.md). | – |
| Rahmen | Hosting: WordPress, MySQL ist nach Bedarf wählbar. Daher dürfen InnoDB und eine aktuelle MySQL-Version vorausgesetzt werden. | – |

Aus E-F3 ergeben sich die neuen offenen Punkte E-F17 bis E-F20 (Abschnitt 5.1).

Dieses Dokument setzt den ersten Arbeitsauftrag aus `AGENTS.md` um:

1. Prüfung auf Widersprüche und offene Punkte (Abschnitt 4)
2. Vorschlag für die Verzeichnisstruktur (Abschnitt 2)
3. Offene Architekturentscheidungen (Abschnitt 5)

Anwendungscode wurde nicht erzeugt. Die Codebeispiele in Abschnitt 3 sind Skizzen, um die Schichtung zu zeigen.

---

## 1. Grundprinzip der Schichtung

```
   WordPress-Frontend        WP-Admin            REST API (DFK-App)
          │                     │                       │
          ▼                     ▼                       ▼
 Controller/Frontend    Controller/Admin         Controller/Rest
          └──────────────┬──────┴───────────────────────┘
                         ▼
                    Validation          ← Eingaben prüfen und bereinigen, gemeinsam für alle Kanäle
                         ▼
                      Service           ← gesamte Geschäftslogik, genau einmal vorhanden
                         ▼
                    Repository          ← einziger Ort mit SQL ($wpdb->prepare)
                         ▼
                 MySQL / MariaDB
```

**Abhängigkeitsregeln (verbindlich):**

| Schicht | Darf verwenden | Darf nicht |
|---|---|---|
| Model | nur andere Models, Enums, Value Objects | WordPress-Funktionen, `$wpdb`, Services |
| Repository | `$wpdb`, Models | Services, Controller, Ausgabe |
| Service | Repositories, andere Services, Models | `$_POST`/`$_GET`, Ausgabe, direktes SQL |
| Controller | Validation, Services, Renderer | Repositories, SQL, Geschäftsregeln |
| View (Template) | übergebene ViewModels, Escaping-Funktionen | Datenbank, Services, Geschäftslogik |
| Integration | Services, WordPress-Hooks | eigene Geschäftslogik |

Die Abhängigkeiten werden ohne Framework per Konstruktor übergeben. `Plugin.php` ist die einzige Stelle, an der Objekte erzeugt und verdrahtet werden (Composition Root). Ein DI-Container ist bei der überschaubaren Größe nicht nötig.

Namespace: `DFK\Events\`

---

## 2. Verzeichnisstruktur

Das Repository ist gleichzeitig das Plugin-Verzeichnis `wp-content/plugins/dfk-events/`.

```
dfk-events/
├── dfk-events.php                 Plugin-Header, Konstanten, Autoloader laden, Plugin::boot()
├── autoload.php                   Eigener PSR-4-Autoloader (ca. 20 Zeilen, siehe E-T3)
├── uninstall.php                  Datenbereinigung gemäß Einstellung (siehe E-F14)
├── composer.json                  Nur Dev-Werkzeuge: PHPUnit, PHPCS + WPCS, PHPStan
├── phpcs.xml.dist
├── phpunit.xml.dist
│
├── src/
│   ├── Plugin.php                 Composition Root: Objekte erzeugen, register() aufrufen
│   │
│   ├── Core/
│   │   ├── Activator.php          Migrationen ausführen, Capabilities anlegen, Rewrites
│   │   ├── Deactivator.php        Cron-Jobs entfernen, Rewrites
│   │   ├── Capabilities.php       Zentrale Definition aller Rechte
│   │   ├── Assets.php             CSS/JS nur auf Seiten laden, die sie brauchen
│   │   └── Settings.php           Typisierter Zugriff auf Plugin-Optionen
│   │
│   ├── Model/                     Fachobjekte – reines PHP, kein WordPress, kein SQL
│   │   ├── Event.php              Veranstaltung inkl. Zusatzfeld-Definition
│   │   ├── Venue.php              Veranstaltungsort mit Koordinaten
│   │   ├── Registration.php       Eine Person einer Buchung (Anmelder oder Gast)
│   │   ├── Booking.php            Zusammenfassung der Registrations einer booking_id
│   │   └── Enum/                  EventStatus, Format, RegistrationStatus, AttendeeType, FieldType
│   │
│   ├── Repository/                Einziger Ort mit SQL
│   │   ├── EventRepository.php
│   │   ├── VenueRepository.php
│   │   └── RegistrationRepository.php
│   │
│   ├── Service/                   Geschäftslogik
│   │   ├── EventService.php           Anlegen, Ändern, Übersicht/Filter, Regel „online → DFK“
│   │   ├── BookingService.php         Buchen, Stornieren, Ändern, Nachrücken; Kapazität, Warteliste, Token
│   │   ├── FormService.php            Zusatzfelder validieren, sensible Werte verschlüsseln
│   │   ├── NotificationService.php    Mails bei Buchung, Storno, Änderung (ADR 0008)
│   │   ├── GeoService.php             Geocoding beim Speichern, Luftlinie, Umkreis
│   │   ├── IcsService.php             Kalenderexport
│   │   └── PurgeService.php           Bereinigung nach MV-Sync bzw. 30 Tagen (ADR 0005)
│   │
│   ├── Validation/
│   │   └── BookingRequest.php     Geprüfte Buchungsanfrage, gemeinsam für Formular und REST
│   │
│   ├── Controller/
│   │   ├── Frontend/
│   │   │   ├── EventListController.php      Übersicht
│   │   │   ├── EventDetailController.php    Detailseite /event/<slug>/
│   │   │   └── BookingController.php        Formular, Verwaltungslink, Spamschutz
│   │   ├── Admin/
│   │   │   ├── EventEditController.php      Metabox: Termin, Ort, Kapazität, Regeln, Zusatzfelder
│   │   │   ├── VenueController.php          Orte verwalten, Koordinaten prüfen/korrigieren
│   │   │   └── RegistrationListController.php  Anmeldungen, Warteliste, Nachrücken, Export
│   │   └── Rest/                       /wp-json/dfk-events/v1/
│   │       ├── EventsController.php         öffentlich
│   │       └── BookingsController.php       Anlegen öffentlich, Verwalten per Login oder Token
│   │
│   ├── View/
│   │   └── Renderer.php           Lädt Templates, erlaubt Override im Child-Theme
│   │
│   ├── Integration/               WordPress-Anbindung, keine Geschäftslogik
│   │   ├── EventPostType.php          CPT dfk_event, Rewrite /event/
│   │   ├── UnitTaxonomy.php           Taxonomie dfk_unit (Veranstalter)
│   │   ├── TemplateLoader.php         template_include, unabhängig von Elementor
│   │   └── Cron.php                   tägliche Bereinigung
│   │
│   └── Database/
│       ├── Migrator.php           Versionierte Migrationen inkl. MV-Views
│       └── Transaction.php        START TRANSACTION / COMMIT / ROLLBACK um $wpdb
│
├── templates/                     Views – überschreibbar unter <child-theme>/dfk-events/
│   ├── archive-event.php
│   ├── single-event.php
│   ├── partials/
│   │   ├── filter-bar.php
│   │   ├── event-card.php
│   │   ├── event-list-item.php
│   │   ├── availability.php
│   │   ├── booking-form.php
│   │   └── attendee-fieldset.php
│   ├── admin/
│   └── emails/
│
├── assets/                        Ohne Build-Schritt, Vanilla JS als ES-Module
│   ├── css/  frontend.css, admin.css
│   └── js/   event-filter.js, location-picker.js, booking-form.js, admin-fields.js
│
├── languages/  dfk-events.pot
│
├── tests/
│   ├── Unit/          Services, Regeln, Distanzberechnung – ohne WordPress
│   ├── Integration/   Repositories, Transaktionen, REST – mit WP-Testsuite
│   └── bootstrap.php
│
└── docs/
    ├── architektur.md         dieses Dokument
    ├── datenbankschema.md     folgt, Freigabe erforderlich
    ├── api.md                 folgt, Freigabe erforderlich
    └── adr/                   eine Datei je freigegebener Architekturentscheidung
```

**Begründungen zu einzelnen Punkten**

- **Model ohne WordPress:** Dadurch lassen sich Kapazitäts-, Buchungs- und Wartelistenregeln mit schnellen Unit-Tests ohne Datenbank prüfen. Genau diese Logik ist laut AGENTS.md §15 geschäftskritisch.
- **Validation als eigene Schicht:** Formular und REST-API liefern dieselbe `BookingRequest` an den `BookingService`. So gibt es keine zweite Prüflogik für die App.
- **ViewModels:** Templates bekommen fertig formatierte Werte (z. B. „noch 3 Plätze“, Datum in deutscher Schreibweise). So bleiben sie frei von Logik, wie in §3 und §4 gefordert.
- **Integration getrennt von Controllern:** Controller verarbeiten Anfragen. Die Integration verbindet WordPress, Rank Math, WP-Members und Cron. Fällt ein Fremd-Plugin weg, ist nur eine Datei betroffen.
- **Kein Build-Schritt:** Moderne Browser verstehen ES-Module direkt. TypeScript würde einen Build erfordern und ist laut §3 nur bei begründetem Bedarf vorgesehen.

---

## 3. Beispiel: Ein Buchungsvorgang durch alle Schichten

Skizze, kein freigegebener Code:

```php
// Controller/Rest/BookingsController.php  (Frontend-Controller analog)
public function create( WP_REST_Request $request ) {
    // User-ID ist 0 bei Buchung ohne Login – das ist der Normalfall, kein Sonderfall (ADR 0002)
    $booking_request = $this->validator->booking( $request->get_json_params(), get_current_user_id() );
    $result          = $this->bookings->book( $booking_request );   // derselbe Service wie im Frontend
    return rest_ensure_response( BookingResource::from( $result ) );
}

// Service/BookingService.php
public function book( BookingRequest $req ): BookingResult {
    return $this->transaction->run( function () use ( $req ) {
        $event = $this->events->lockForUpdate( $req->eventId );   // SELECT … FOR UPDATE
        $this->rules->assertBookable( $event, $req );             // Frist, Teilnehmerarten, Max. pro Buchung
        $free  = $this->capacity->freeSeats( $event );            // zählt Teilnehmer, nicht Buchungen
        // Wer einen Platz erhält und wer auf die Warteliste kommt, entscheidet
        // der WaitlistService – Regeln noch offen (E-F9).
        …
    } );
}
```

Die Sperre auf die Eventzeile bewirkt, dass zwei gleichzeitige Buchungen für dasselbe Event nacheinander abgearbeitet werden. Buchungen für unterschiedliche Events blockieren sich nicht gegenseitig.

---

## 4. Prüfung der AGENTS.md: Widersprüche und Lücken

Echte Widersprüche gibt es keine. Folgende Stellen sind aber mehrdeutig oder unvollständig:

| # | Abschnitt | Befund |
|---|---|---|
| W1 | §5 / §10 | Es gibt zwei Arten von Status: den **Veröffentlichungsstatus** des Beitrags (Entwurf, veröffentlicht) und den **Veranstaltungsstatus** (findet statt, abgesagt, ausgebucht). Das sollte klar getrennt werden. „Ausgebucht“ sollte berechnet und nicht gespeichert werden. |
| W2 | §7 | „Mehrfachanmeldungen“ kann zweierlei heißen: (a) mehrere Teilnehmer in *einer* Buchung oder (b) *mehrere Buchungen* derselben Person für dasselbe Event. Die Anforderung „max. Teilnehmer je Buchung“ deckt (a) bereits ab. Gemeint ist vermutlich (b). |
| W3 | §10 / §11 | **Regionalgruppe** und **Veranstalter** werden angezeigt bzw. gefiltert, kommen im Datenmodell (§5) aber nicht vor. |
| W4 | §10 | „Bildbasierte **Kartenansicht**“ – Kachel-/Card-Ansicht oder Landkarte? Eine Landkarte würde einen Kartendienst (Datenschutz, Kosten) erfordern. |
| W5 | §6 | Für Online-Veranstaltungen fehlt der Zugangslink. Offen ist auch, ob er nur angemeldeten Teilnehmern angezeigt wird. Hybride Veranstaltungen (vor Ort + online) sind nicht erwähnt. |
| W6 | §1 | ~~Bezahlung kommt nirgends vor.~~ **Geklärt:** keine Bezahlung, nur RSVP (E-F1). |
| W7 | §1 | The Events Calendar **Pro** bietet wiederkehrende Veranstaltungen und Serien. Sie sind nicht erwähnt. |
| W8 | §11 / §16 | Pfad `/event/` kollidiert mit The Events Calendar, solange beide auf `dev.dfk.eu` aktiv sind. |
| W9 | §11 | Sind auf dfk.eu Seiten-Caches aktiv (Plugin, Server, CDN), zeigen gecachte Detailseiten veraltete Platzzahlen, und Nonces im Formular laufen ab. Siehe E-T8. |
| W10 | §12 | Elementor Pro Theme Builder kann eigene Single-/Archiv-Templates erzwingen. Es muss festgelegt werden, dass für `dfk_event` keine Elementor-Bedingungen greifen. |
| W11 | §7 | Wer darf eine Buchung oder einzelne Teilnehmer stornieren, und bis wann? Nur die anmeldende Person, auch der Teilnehmer selbst, nur Admins? |
| W12 | – | Rechte und Rollen fehlen: Wer darf Events anlegen und Teilnehmerlisten sehen? Dürfen Regionalgruppen-Verantwortliche nur ihre eigenen Events verwalten? |
| W13 | – | E-Mail-Benachrichtigungen (Inhalt, Absender, Versandweg) sind nur als Schlagwort erwähnt. |
| W14 | §8 | Löschfristen sollen konfigurierbar sein, es fehlt aber der Bezugspunkt (z. B. X Tage nach Veranstaltungsende) und was gelöscht bzw. anonymisiert wird. |

---

## 5. Offene Architekturentscheidungen

Jede Entscheidung erhält eine ID. Freigegebene Entscheidungen werden als eigene Datei unter `docs/adr/` festgehalten. Meine Empfehlung ist jeweils angegeben, die Entscheidung trifft der Captain.

### 5.1 Fachliche Entscheidungen

| ID | Frage | Optionen | Empfehlung |
|---|---|---|---|
| E-F1 | Bezahlte Veranstaltungen in Version 1? | – | ✅ **Entschieden:** keine Bezahlung, nur RSVP |
| E-F2 | Wiederkehrende Veranstaltungen? | nein / einfache Kopierfunktion / echte Serien | Kopierfunktion („Event duplizieren“) in V1, echte Serien später |
| E-F3 | Dürfen Gäste ohne Login buchen? | – | ✅ **Entschieden:** Buchen ist immer ohne Login möglich, ein Login füllt nur Daten vor |
| E-F4 | Wie wird die Teilnehmerart „Mitglied“ festgestellt, wenn ohne Login gebucht wird? | – | ✅ **Entschieden:** Je Teilnehmer ein Dropdown „Mitglied / Nichtmitglied“, Selbstauskunft ohne Prüfung, keine Mitgliedsnummer (ADR 0003) |
| E-F5 | Regionalgruppe – wie abbilden? | – | ✅ **Entschieden:** Taxonomie `dfk_unit` mit Typ DFK / Regionalgruppe / Netzwerk, genau eine je Event; Kennung aus der MV (`RegionalgruppeID` bzw. Netzwerk-Kürzel). Online-Events → immer DFK |
| E-F6 | Veranstaltungsstatus | – | ✅ **Entschieden:** findet statt / abgesagt / verschoben; „ausgebucht“ und „Anmeldung geschlossen“ werden berechnet |
| E-F7 | Veranstalter – wie abbilden? | – | ✅ **Entschieden:** Auswahl aus `T_Veranstalter` (MV), Kennung `InterneID`; ersetzt den Freitext |
| E-F8 | Kartenansicht (W4) | Kacheln / Landkarte / beides | Kacheln in V1; Landkarte später, da Kartendienst nötig |
| E-F9 | Wartelisten-Regeln | automatisch / manuell nachrücken; Gruppen ganz oder teilweise; Reservierungsfrist | Vorschlag folgt als eigenes Papier, sobald E-F3 und E-F11 geklärt sind |
| E-F10 | Kontingente je Teilnehmerart? | eine Gesamtkapazität / getrennte Kontingente Mitglieder/Gäste | Gesamtkapazität in V1, Datenmodell so anlegen, dass Kontingente nachrüstbar sind |
| E-F11 | Stornierung (W11) | wer, bis wann, mit Benachrichtigung? | Anmeldende Person bis Anmeldeschluss – per Login **oder** per Verwaltungslink aus der Bestätigungsmail (E-F18) –, danach nur Admins |
| E-F12 | Rollen und Rechte (W12) | nur Admins / Redakteure / Regionalgruppen-Verantwortliche mit eingeschränkten Rechten | Eigene Capabilities von Anfang an, Zuordnung zu Rollen konfigurierbar |
| E-F13 | Online-Zugangslink (W5) | öffentlich / nur für bestätigte Teilnehmer | Nur für bestätigte Teilnehmer (Mail + „Meine Buchungen“) |
| E-F14 | Was passiert bei Deinstallation? | alles löschen / Daten behalten | Daten standardmäßig behalten, Löschen nur per ausdrücklicher Einstellung |
| E-F15 | Löschfristen (W14) | – | ✅ **Entschieden:** Löschung 30 Tage nach dem Veranstaltungstermin (ADR 0005) |
| E-F25 | Welches CRM übernimmt die Daten? | – | ✅ **Geklärt:** DFK-Mitgliederverwaltung (Eigenentwicklung, Access + SQL Server). Ist-Analyse: [mv-schnittstelle.md](mv-schnittstelle.md) |
| E-F26 | Wie übernimmt die MV die Daten? | – | ✅ **Entschieden: A für V1** (MV liest heute schon per MySQL-ODBC), B mit der .NET-MV ab 2027. Details: [mv-schnittstelle.md](mv-schnittstelle.md) |
| E-F27 | Wer löst die Löschung aus? | – | ✅ **Entschieden (18.11.2026): nur die Frist** – Bereinigung 30 Tage nach Veranstaltungsende, keine Sync-Bestätigung |
| E-F28 | Umfang der Löschung | – | ✅ **Entschieden:** alle personenbezogenen Buchungsdaten (anmeldende Person, Teilnehmer, Zusatzfelder) vollständig löschen. Keine Statistik im Plugin – Auswertungen macht die MV-Software |
| E-F16 | Migration | welche Daten aus TEC/ETP übernehmen (vergangene Events? alte Anmeldungen?) | Separates Migrationskonzept; alte Anmeldungen nur, soweit Löschfristen es erlauben |
| E-F17 | Bestätigung und Missbrauchsschutz bei Buchung ohne Login | (a) sofort bestätigt + Mail / (b) Double-Opt-in: Platz vorläufig reserviert, verfällt ohne Klick / (c) pro Event wählbar | **(a) als Standard** mit Honeypot, Rate-Limit je IP/E-Mail und Plausibilitätsprüfungen; **(b) pro Event zuschaltbar**, z. B. bei knappen Plätzen. Ein CAPTCHA nur, falls nötig (externer Dienst, Datenschutz) |
| E-F18 | Wie verwaltet jemand ohne Konto seine Buchung? | Verwaltungslink mit geheimem Token in der Mail / Code + E-Mail-Adresse / nur über Admins | **Verwaltungslink mit Token**: Buchung ansehen, einzelne Teilnehmer oder alles stornieren. Token zufällig, nur gehasht gespeichert, verfällt nach Eventende |
| E-F19 | Mehrfachanmeldung ohne Login erkennen (W2) | über E-Mail-Adresse / gar nicht | **Über die E-Mail-Adresse der anmeldenden Person** (normalisiert); bei Login zusätzlich über die User-ID |
| E-F21 | Welche Personendaten werden je Teilnehmer erfasst (auch für Gäste)? | – | ✅ **Entschieden:** Vorname, Nachname, Mitglied/Nichtmitglied, E-Mail – alle Pflicht, für jeden Teilnehmer (ADR 0004) |
| E-F22 | Dürfen mehrere Teilnehmer einer Buchung dieselbe E-Mail haben (z. B. Ehepaar mit gemeinsamer Adresse)? | erlaubt / jede E-Mail nur einmal je Event | **Erlaubt** – sonst scheitern Paare mit gemeinsamer Adresse |
| E-F23 | Bekommt jeder Teilnehmer eine eigene Mail? | – | ✅ **Entschieden (vorerst):** Nur die anmeldende Person erhält eine Bestätigung mit der Liste aller angemeldeten Personen. Mails an einzelne Teilnehmer sind später nachrüstbar (`NotificationService`) |
| E-F24 | Umgang mit sensiblen Zusatzfeldern (z. B. Personalausweisnummer) | – | ✅ **Entschieden:** **Kennzeichen „sensibel“ je Feld**. Löschfrist in Tagen nach Eventende als Einstellung, Standardwert noch festzulegen. Solche Werte: nur für Berechtigte mit eigener Capability sichtbar, nicht in der REST-API für die App, nicht in Standard-Exporten (nur per ausdrücklichem Export), nicht in Mails und Logs, verschlüsselt gespeichert, automatisch gelöscht X Tage nach Eventende |
| E-F20 | Gastbuchungen und WP-Konten | Buchungen mit gleicher E-Mail automatisch unter „Meine Buchungen“ zeigen / nur Buchungen, die eingeloggt erstellt wurden | Nur eingeloggt erstellte Buchungen; die automatische Zuordnung per E-Mail nur, wenn die E-Mail des Kontos verifiziert ist |

### 5.2 Technische Entscheidungen

| ID | Frage | Optionen | Empfehlung |
|---|---|---|---|
| E-T1 | Mindestversion PHP | 8.0 / 8.1 / 8.2 | **8.1** (native Enums, `readonly`). Vorher prüfen, was dev und Produktion tatsächlich bereitstellen |
| E-T2 | Mindestversion WordPress | – | Aktuelle Version minus zwei Hauptversionen; an Produktion ausrichten |
| E-T3 | Autoloading | Composer-Autoloader (vendor/ mit ausliefern) / eigener PSR-4-Autoloader | **Eigener Autoloader**; Composer nur für Dev-Werkzeuge. So gibt es keine Laufzeitabhängigkeiten und kein vendor/ auf dem Server |
| E-T4 | Transaktionen und Sperren | `SELECT … FOR UPDATE` auf Eventzeile / Zählerspalte mit atomarem UPDATE / Advisory Locks | **FOR UPDATE auf Eventzeile**. InnoDB ist laut Rahmenentscheidung verfügbar; Mindestversion vorschlagsweise MySQL 8.0 / MariaDB 10.6 |
| E-T5 | Verknüpfung CPT ↔ `dfk_events` | `post_id` als Primärschlüssel / eigene ID + eindeutige `post_id` | **`post_id` als Schlüssel (1:1)**; Verhalten bei Papierkorb, endgültigem Löschen und Duplizieren festlegen |
| E-T6 | Geocoding der Veranstaltungsorte | Nominatim (OSM) / Google / manuelle Eingabe | Einmalig beim Speichern per Nominatim, manuelle Korrektur möglich |
| E-T7 | Standortsuche der Nutzer (PLZ/Ort) | externer Dienst bei jeder Suche / lokale PLZ-Tabelle | **Lokale PLZ-Koordinatentabelle** – keine Weitergabe von Nutzereingaben, schnell, keine Limits. Browser-Standort wird nur im Browser verwendet und nicht gespeichert |
| E-T8 | Seiten-Caching (W9) | Detailseiten nicht cachen / Platzzahl und Nonce per REST nachladen | **Nachladen per REST** – die Seite bleibt cachebar, Platzzahl ist immer aktuell |
| E-T9 | Formularversand Frontend | klassisch `admin-post.php` / `fetch()` gegen eigene REST-Route | **REST mit Fallback** auf normalen Formularversand, damit es auch ohne JS funktioniert |
| E-T10 | Authentifizierung der App | Application Passwords / JWT-Plugin / OAuth2 | Gesondert entscheiden, wenn App-Integration ansteht; in V1 nur Cookie + Nonce. API-Controller so bauen, dass das Verfahren austauschbar ist. Das Anlegen von Buchungen braucht ohnehin keine Authentifizierung (ADR 0002) |
| E-T11 | Umkreissuche in SQL | Haversine in SQL mit Bounding-Box-Vorfilter / Berechnung in PHP | **Bounding Box in SQL** (Index auf lat/lng), genaue Distanz danach berechnen |
| E-T12 | Zeitzonen | Speicherung lokal / UTC | **UTC in der Datenbank**, Anzeige in Europe/Berlin |
| E-T13 | Rank Math | Rank Math gibt Event-Schema aus / Plugin gibt JSON-LD aus | Plugin liefert die Daten, nur eine Stelle gibt Schema aus – keine Doppelung |
| E-T14 | Admin-Oberfläche für Events | klassische Metaboxen / Gutenberg-Sidebar (benötigt React) | **Metaboxen** – kein React im Plugin, passt zu §3 |
| E-T15 | Tests | PHPUnit + WP-Testsuite / zusätzlich Brain Monkey für Unit-Tests | PHPUnit für alles; Unit-Tests ohne WordPress dank WP-freier Models und Services |
| E-T16 | Paralleler Betrieb auf dev (W8) | TEC deaktivieren / anderer Slug während der Tests | Während der Entwicklung `/dfk-event/`, Umstellung auf `/event/` bei Migration |
| E-T17 | Logging | `error_log` / eigene Tabelle / WooCommerce-artiger Datei-Logger | Eigener schlanker Logger auf Basis `error_log`, ohne personenbezogene Daten |

---

## 6. Vorgeschlagene nächste Schritte

1. Captain entscheidet die noch offenen fachlichen Punkte (E-F2, E-F5 bis E-F20) oder benennt, wer sie klären soll.
2. Technische Rahmendaten von dev und Produktion erheben: PHP- und WordPress-Version, eingesetztes Caching.
3. Auf dieser Basis: Entwurf `docs/datenbankschema.md` zur Freigabe.
4. Danach: Entwurf `docs/api.md` zur Freigabe.
5. Erst dann: Grundgerüst des Plugins (Bootstrap, Autoloader, Migrator, CPT) auf einem Feature-Branch.
