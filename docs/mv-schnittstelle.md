# Schnittstelle DFK Events ↔ DFK-Mitgliederverwaltung (MV)

Stand: 17.11.2026 · Status: **Freigegeben (Captain, 17.11.2026)** – Variante A für V1, Sync-Bestätigung durch die MV

Quelle der Analyse: Repository `DFK` (Access-Frontend „DFK Mitgliederverwaltung“, Backend SQL Server), Modul `modules/INet.bas` (`Termine_Import_V2`, `Termine_Anmeldungen_V2`), Tabellen `T_Termine`, `T_Termine_Anmeldungen`. Es wurden nur Quelltext und Schema gelesen, keine Daten.

---

## 1. Ist-Zustand

Die MV liest **direkt per MySQL-ODBC** (DSN `DFKWP`) aus der WordPress-Datenbank. Ausgelöst wird das manuell über den Ribbon-Button „Anmeldungen synchronisieren“ (`GetEventData`).

**Veranstaltungen** (`Termine_Import_V2` → `T_Termine`):

| MV-Feld | Quelle in WordPress (The Events Calendar) |
|---|---|
| TerminID | `wp_posts.ID` (`post_type = 'tribe_events'`, nur `publish`) |
| Titel | `post_title` |
| Startdatum / Enddatum | `_EventStartDate` / `_EventEndDate` |
| Veranstalter | `tribe_organizer.post_title` |
| Ort / OrtLocation | Venue-Titel / `_VenueCity` (für Heatmap) |
| Abgesagt | `_tribe_events_status` ≠ „online“ |
| Anmeldeschluss | `_ticket_end_date` des RSVP-Tickets |
| MaxTeilnehmer | `_tribe_ticket_capacity` |

**Anmeldungen** (`Termine_Anmeldungen_V2` → `T_Termine_Anmeldungen`):

| MV-Feld | Quelle / Logik |
|---|---|
| Termin_AnmeldungId | ID des Attendee-Posts (`tribe_rsvp_attendees`) |
| TerminID | `_tribe_rsvp_event` |
| HauptbuchungsID | `_tribe_rsvp_order` → entspricht unserer **Buchung** |
| GastStatus | `H` = Hauptgast (erster der Bestellung), `B` = Begleitperson |
| Vorname, Nachname | `_tribe_tickets_meta` (PHP-serialisiert) |
| EMail | `begleitperson-e-mail` aus Meta, sonst E-Mail der Bestellung |
| Anmeldedatum | `post_date` |
| **Mitgliedsstatus, Mitgliedsnummer, Mitgliedsart, Regionalgruppe** | **werden in der MV ermittelt**: Abgleich der E-Mail mit privater oder dienstlicher Mail in `Q_Alles` plus Nachname. Kein Treffer → „Interessent“ |

Inkrementell: Es werden nur Datensätze mit `post_modified >= T_Konstanten.DatTerminUpdate` gelesen. Danach berechnet die SP `dbo.usp_RecalcTermineCounters` die Teilnehmerzahlen.

## 2. Was das für DFK Events bedeutet

1. **Das Datenmodell passt bereits.** Buchung ↔ `HauptbuchungsID`, Teilnehmer ↔ `Termin_AnmeldungId`, anmeldende Person ↔ `GastStatus = 'H'`. Die Entscheidung „E-Mail für jeden Teilnehmer Pflicht“ (ADR 0004) ist genau das, was die MV für den Mitgliederabgleich braucht.
2. **Regionalgruppe und Mitgliedsart der Teilnehmer kommen aus der MV,** nicht von der Webseite. DFK Events muss sie nicht liefern. Die Selbstauskunft Mitglied/Nichtmitglied (ADR 0003) ist eine zusätzliche Angabe, die MV ermittelt den Status weiterhin selbst.
3. **Die Regionalgruppe einer Veranstaltung** (E-F5) wird heute nicht in die MV übernommen. Sie ist für DFK Events ein reines Filterkriterium der Webseite – es sei denn, die MV soll sie künftig auch auswerten.
4. **Die MV-Abfragen müssen beim Umstieg ohnehin angepasst werden,** weil es `tribe_events`/`tribe_rsvp_attendees` dann nicht mehr gibt. Das ist die Gelegenheit, eine saubere Schnittstelle zu definieren.

## 3. Vorschlag für die neue Schnittstelle

### Variante A – Datenbank-Views als Vertrag (Empfehlung für V1)

- DFK Events legt versionierte, nur lesbare **Views** an: `{prefix}dfk_mv_veranstaltungen_v1`, `{prefix}dfk_mv_orte_v1` und `{prefix}dfk_mv_anmeldungen_v1`. Sie bilden die künftigen MV-Tabellen `T_Veranstaltungen` / `T_Veranstaltungsorte` / `T_Veranstaltungen_Anmeldungen` nahezu 1:1 ab (ADR 0007, Spalten in [datenbankschema.md](datenbankschema.md)).
- Die MV bekommt einen **eigenen MySQL-Benutzer mit `SELECT` nur auf diese Views** – kein Zugriff mehr auf `wp_posts`, `wp_users` usw.
- Interne Tabellen können sich ändern, solange die Views stabil bleiben. Eine neue Version erhält eine neue View (`_v2`).
- **Vorteil:** Die MV behält ihren ODBC-Weg, die Anpassung in VBA ist klein. Kein PHP-serialisiertes Parsen mehr.

### Variante B – REST-Endpunkt

- `GET /wp-json/dfk-events/v1/mv/attendees?changed_since=…` mit technischem Benutzer.
- **Vorteil:** sauberste Trennung, kein offener Datenbankzugang; passt zur geplanten .NET-Migration der MV ab 2027.
- **Nachteil:** HTTP- und JSON-Verarbeitung in VBA ist aufwendiger.

**Empfehlung:** A für den Start, B, sobald die .NET-MV den Teil „Veranstaltungen“ übernimmt. Beide Varianten lesen dieselben Services/Daten, die Entscheidung ist also nicht endgültig.

### Rückschreiben durch die MV (ADR 0009, Vorschlag)

Die MV darf ausgewählte Spalten der Anmeldungen direkt aktualisieren: Vor- und Nachname (Korrekturen), Mitgliedsnummer, geprüfter Mitgliedsstatus und Zeitpunkt der Prüfung. Umgesetzt über spaltengenaue MySQL-Rechte; alle anderen Daten bleiben für die MV nur lesbar.

### Stornos und Löschung

- Heute sieht die MV gelöschte Anmeldungen in WordPress nicht. Neu: Stornierte Teilnehmer bleiben bis zur Bereinigung mit **Status „storniert“** und neuem `updated_at` in der View, damit die MV sie nachziehen kann.
- Nach der Bereinigung (ADR 0005) fehlen die Datensätze in der View. Die MV darf daraus **nicht** auf ein Storno schließen. Vorschlag: Bereinigung erst, wenn das Event vorbei ist **und** die MV synchronisiert hat (E-F27).
- Übernahmebestätigung (E-F27): Die MV schreibt nach erfolgreichem Lauf einen Zeitstempel in eine kleine Tabelle `{prefix}dfk_mv_sync` (eigenes `INSERT`-Recht nur dafür). DFK Events bereinigt nur Events, deren Ende vor diesem Zeitstempel liegt. Die 30-Tage-Frist bleibt als Sicherheitsnetz.

## 4. Beobachtungen im MV-Code (nicht Teil dieses Projekts, aber relevant)

1. **SQL-Injection-Risiko:** `Termine_Anmeldungen_V2` setzt Vorname, Nachname und E-Mail aus dem Webformular per Stringverkettung in SQL für SQL Server ein. Schon ein Name wie „O'Brien“ bricht den Import ab. Da Buchungen künftig **ohne Login öffentlich** möglich sind, könnte ein präparierter Name SQL auf dem MV-Server ausführen. DFK Events validiert Eingaben, aber die MV-Seite muss **parametrisierte Abfragen** verwenden. Gleiches gilt für den Abgleich gegen `Q_Alles`.
2. **Zugangsdaten im Quelltext:** `INet.bas` (Sub `TestMySQLConnection`) enthält in auskommentiertem Code Server, Benutzer und Passwort einer MySQL-Datenbank. Einschätzung des Captains: Die Datenbank liegt im lokalen Netz hinter Firewalls, das Risiko wird akzeptiert. Hinweis: Der dort genannte Hostname sieht nach einem externen Hosting-Server aus – das sollte einmal gegengeprüft werden.
3. Die WordPress-Datenbank ist von außen per MySQL erreichbar. Mit Variante A sollte der Zugang auf den neuen, eingeschränkten Benutzer und möglichst auf feste IP-Adressen begrenzt werden.
