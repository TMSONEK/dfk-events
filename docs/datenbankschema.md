# DFK Events – Datenbankschema (Entwurf)

Stand: 18.11.2026 · Status: **Freigegeben (Captain, 18.11.2026)** inkl. ADR 0009

Grundlage: [architektur.md](architektur.md), ADR 0001–0008, [mv-schnittstelle.md](mv-schnittstelle.md).

**Leitlinie:** 100–120 Veranstaltungen pro Jahr, nur Deutschland, keine Migration (ADR 0006). **Drei Tabellen** – Veranstaltungen, Orte, Anmeldungen –, Aufbau gleich wie künftig in der MV (ADR 0007).

---

## 1. Allgemeine Regeln

- MySQL 8.0 / MariaDB 10.6, InnoDB, Zeichensatz aus `$wpdb->get_charset_collate()`.
- Tabellennamen mit WordPress-Präfix, im Folgenden `wp_`.
- Alle Zeiten in deutscher Ortszeit.
- Status- und Typwerte als `VARCHAR`, erlaubte Werte legt ein PHP-Enum fest.
- `created_at` / `updated_at` in jeder Tabelle (`updated_at` für den MV-Abgleich).
- Schemaänderungen über versionierte Migrationen, Stand in Option `dfk_events_db_version`.

## 2. Übersicht

```
wp_posts (post_type = dfk_event)      Titel, Beschreibung, Bild, Permalink, Veröffentlichung
   │ 1:1
   ▼
wp_dfk_events                         eine Zeile je Veranstaltung  ≙ MV T_Veranstaltungen
   │   └── n:1 ──▶ wp_dfk_venues      Orte mit Geokoordinaten      ≙ MV T_Veranstaltungsorte
   │ 1:n
   ▼
wp_dfk_registrations                  eine Zeile je Person          ≙ MV T_Veranstaltungen_Anmeldungen
                                      zusammengefasst über booking_id

Taxonomie dfk_unit                    Veranstalter (aus MV T_Veranstalter)
```

## 3. Tabellen

### 3.1 `wp_dfk_events` – Veranstaltungen

| Spalte | Typ | Bemerkung |
|---|---|---|
| `post_id` | BIGINT UNSIGNED **PK** | = `wp_posts.ID` |
| `starts_at` | DATETIME NOT NULL | |
| `ends_at` | DATETIME NOT NULL | |
| `all_day` | TINYINT(1) NOT NULL DEFAULT 0 | |
| `status` | VARCHAR(20) NOT NULL DEFAULT 'scheduled' | `scheduled`, `cancelled`, `postponed` |
| `format` | VARCHAR(10) NOT NULL DEFAULT 'onsite' | `onsite`, `online`, `hybrid` |
| `venue_id` | BIGINT UNSIGNED NULL | → `wp_dfk_venues.id`; NULL bei reinen Online-Veranstaltungen |
| `online_url` | VARCHAR(2048) NULL | nur für angemeldete Teilnehmer sichtbar |
| `registration_enabled` | TINYINT(1) NOT NULL DEFAULT 1 | |
| `registration_closes_at` | DATETIME NULL | Anmeldeschluss |
| `capacity` | INT UNSIGNED NULL | NULL = unbegrenzt |
| `max_per_booking` | SMALLINT UNSIGNED NOT NULL DEFAULT 5 | |
| `allow_members` | TINYINT(1) NOT NULL DEFAULT 1 | zulässige Teilnehmerarten |
| `allow_non_members` | TINYINT(1) NOT NULL DEFAULT 1 | |
| `allow_multiple_bookings` | TINYINT(1) NOT NULL DEFAULT 0 | mehrere Buchungen je E-Mail |
| `waitlist_enabled` | TINYINT(1) NOT NULL DEFAULT 1 | |
| `extra_fields` | JSON NULL | Definition der Zusatzfelder (siehe 3.3) |
| `purged_at` | DATETIME NULL | Zeitpunkt der Bereinigung |
| `created_at`, `updated_at` | DATETIME NOT NULL | |

Index: `(starts_at)`. Der Veranstalter hängt als Taxonomie-Term am Beitrag (3.6).

### 3.2 `wp_dfk_venues` – Veranstaltungsorte

| Spalte | Typ | Bemerkung |
|---|---|---|
| `id` | BIGINT UNSIGNED **PK** AUTO_INCREMENT | |
| `name` | VARCHAR(255) NOT NULL | |
| `street` | VARCHAR(255) NULL | Straße und Hausnummer |
| `postal_code` | VARCHAR(5) NULL | |
| `city` | VARCHAR(255) NULL | |
| `lat` | DECIMAL(9,6) NULL | einmalig beim Speichern ermittelt, manuell korrigierbar |
| `lng` | DECIMAL(9,6) NULL | |
| `directions` | TEXT NULL | Anfahrtsinformationen |
| `is_active` | TINYINT(1) NOT NULL DEFAULT 1 | statt Löschen |
| `created_at`, `updated_at` | DATETIME NOT NULL | |

Orte werden wiederverwendet; die Koordinaten liegen damit an genau einer Stelle. Sie dienen der Umkreissuche, der späteren App und – über die MV-View – dem Python-Landkartenskript. Die Umkreissuche rechnet in PHP (Luftlinie), ein Geo-Index ist bei der Datenmenge nicht nötig.

### 3.3 `wp_dfk_registrations` – Anmeldungen (eine Zeile je Person)

| Spalte | Typ | Bemerkung |
|---|---|---|
| `id` | BIGINT UNSIGNED **PK** AUTO_INCREMENT | Startwert siehe S4 |
| `event_id` | BIGINT UNSIGNED NOT NULL | → `wp_dfk_events.post_id` |
| `booking_id` | BIGINT UNSIGNED NOT NULL | fasst eine Buchung zusammen; = `id` der Anmelderzeile |
| `is_booker` | TINYINT(1) NOT NULL DEFAULT 0 | 1 = anmeldende Person (MV `GastStatus` H), 0 = Gast (B) |
| `user_id` | BIGINT UNSIGNED NULL | WP-Benutzer, nur in der Anmelderzeile, NULL ohne Login |
| `first_name` | VARCHAR(100) NOT NULL | |
| `last_name` | VARCHAR(100) NOT NULL | |
| `email` | VARCHAR(254) NOT NULL | Pflicht, normalisiert gespeichert |
| `attendee_type` | VARCHAR(20) NOT NULL | `member`, `non_member` |
| `status` | VARCHAR(20) NOT NULL DEFAULT 'active' | `active`, `cancelled`, `not_attending` (Anmelder kommt selbst nicht) |
| `waitlist` | TINYINT(1) NOT NULL DEFAULT 0 | **Wartelisten-Flag** (ADR 0008) |
| `extra_values` | JSON NULL | Werte der Zusatzfelder; sensible Werte verschlüsselt |
| `mv_member_no` | INT UNSIGNED NULL | **von der MV geschrieben**: zugeordnete Mitgliedsnummer (ADR 0009) |
| `mv_status` | VARCHAR(30) NULL | **von der MV geschrieben**: geprüfter Status, z. B. `Mitglied`, `Interessent` |
| `mv_updated_at` | DATETIME NULL | **von der MV geschrieben**: Zeitpunkt der letzten Rückmeldung |
| `created_at`, `updated_at` | DATETIME NOT NULL | `created_at` = Reihenfolge der Warteliste |

Indizes: `(event_id, status, waitlist)`, `(booking_id)`, `(event_id, email)`, `(updated_at)`.

Kein Verwaltungslink in V1: Stornos laufen über den Mitgliederservice (offene-punkte.md A3). Ein Token-Feld kann bei Bedarf per Migration ergänzt werden.

`attendee_type` bleibt die Selbstauskunft aus dem Formular; `mv_status` ist der von der MV geprüfte Status. Beide stehen nebeneinander, nichts wird überschrieben.

**Belegte Plätze** = Anzahl Zeilen mit `status = 'active'` und `waitlist = 0`.

**Die anmeldende Person kommt nicht selbst:** Ihre Zeile bleibt als Anmelderzeile (Kontakt, Token) bestehen, mit `status = 'not_attending'`; sie zählt nicht als Platz.

**Neue Buchung:** In einer Transaktion wird die Veranstaltung gesperrt (`SELECT … FOR UPDATE`), die Anmelderzeile eingefügt, ihre `id` als `booking_id` gesetzt und die Gäste mit derselben `booking_id` eingefügt. Passt die Buchung nicht vollständig in die freien Plätze, erhält **die ganze Buchung** das Wartelisten-Flag (ADR 0008).

### 3.4 Zusatzfelder als JSON

Definition in `wp_dfk_events.extra_fields`:

```json
[
  { "key": "ausweis", "label": "Personalausweisnummer", "type": "text",
    "scope": "attendee", "required": true, "sensitive": true },
  { "key": "essen", "label": "Essenswunsch", "type": "select",
    "scope": "attendee", "required": false, "options": ["Fleisch", "Vegetarisch"] }
]
```

Werte in `wp_dfk_registrations.extra_values` als `{ "ausweis": "<verschlüsselt>", "essen": "Vegetarisch" }`. Felder mit `scope = booking` stehen nur in der Anmelderzeile.

Regeln (freigegeben 18.11.2026):
- Pflege in der Metabox der Veranstaltung; „Felder aus Veranstaltung X übernehmen“ statt Vorlagen.
- Sensible Werte werden einzeln verschlüsselt (Schlüssel in `wp-config.php`).
- Sobald Anmeldungen vorliegen: `key` und `type` sind fest, `label` ist änderbar; gelöschte Felder werden nur ausgeblendet, vorhandene Werte bleiben erhalten.
- Keine veranstaltungsübergreifenden Abfragen über Zusatzfelder (Auswertungen macht die MV).

### 3.5 Bereinigung

Ein täglicher Cron löscht alle Anmeldungen von Veranstaltungen, deren Ende mehr als **30 Tage** zurückliegt, und setzt `purged_at` (ADR 0005). Keine eigene Tabelle, keine Bestätigung durch die MV.

### 3.6 Veranstalter – Taxonomie `dfk_unit`

Jede Veranstaltung hat genau einen Veranstalter aus der MV-Liste `T_Veranstalter` (einmaliger CSV-Import, danach Pflege im WP-Admin).

| Term-Meta | Bemerkung |
|---|---|
| `mv_veranstalter_id` | = `T_Veranstalter.InterneID`, Pflicht, eindeutig |
| `unit_type` | `dfk`, `regionalgruppe`, `netzwerk`, `kooperation` |
| `active` | inaktive Veranstalter sind für neue Veranstaltungen nicht wählbar |

Reine Online-Veranstaltungen gehören immer dem DFK; hybride dem Veranstalter vor Ort.

## 4. Views für die MV

Die MV liest nur diese Views. Sie bilden die künftigen MV-Tabellen `T_Veranstaltungen`, `T_Veranstaltungsorte` und `T_Veranstaltungen_Anmeldungen` nahezu 1:1 ab.

**`wp_dfk_mv_orte_v1`**

| Spalte | Herkunft |
|---|---|
| `OrtID` | `id` |
| `Name`, `Strasse`, `PLZ`, `Stadt` | |
| `Lat`, `Lng` | Geokoordinaten (für das Python-Landkartenskript) |
| `LastUpdate` | `updated_at` |

**`wp_dfk_mv_veranstaltungen_v1`**

| Spalte | Herkunft |
|---|---|
| `VeranstaltungID` | `post_id` |
| `Titel` | `wp_posts.post_title` |
| `VeranstalterInterneID`, `Veranstalter` | Term-Meta / Term-Name |
| `Startdatum`, `Enddatum` | `starts_at`, `ends_at` |
| `Anmeldeschluss` | `registration_closes_at` |
| `Format` | `format` |
| `OrtID` | `venue_id` |
| `Ort`, `PLZ`, `Stadt` | aus `wp_dfk_venues`, zur Bequemlichkeit mitgeliefert |
| `Status` | `status` |
| `MaxTeilnehmer` | `capacity` |
| `Veroeffentlicht` | `wp_posts.post_status = 'publish'` |
| `LastUpdate` | `updated_at` |

**`wp_dfk_mv_anmeldungen_v1`**

| Spalte | Herkunft |
|---|---|
| `AnmeldungID` | `id` |
| `VeranstaltungID` | `event_id` |
| `BuchungID` | `booking_id` |
| `GastStatus` | `H` / `B` aus `is_booker` |
| `Vorname`, `Nachname`, `EMail` | |
| `TeilnehmerartAngabe` | `Mitglied` / `Nichtmitglied` (Selbstauskunft) |
| `MitgliedNr`, `MVStatus`, `MVGeprueftAm` | `mv_member_no`, `mv_status`, `mv_updated_at` (von der MV zurückgeschrieben) |
| `Status` | `aktiv`, `storniert`, `nimmt_nicht_teil` |
| `Warteliste` | `waitlist` |
| `Zusatzangaben` | `extra_values` als JSON, **ohne** sensible Felder |
| `Anmeldedatum` | `created_at` |
| `LastUpdate` | `updated_at` |

### Rückschreiben durch die MV (ADR 0009)

Der MySQL-Benutzer der MV erhält zusätzlich **spaltengenaue** Rechte auf `wp_dfk_registrations`:

```sql
GRANT SELECT (id, booking_id, event_id) ON wp_dfk_registrations TO 'dfk_mv'@'<host>';
GRANT UPDATE (first_name, last_name, mv_member_no, mv_status, mv_updated_at)
      ON wp_dfk_registrations TO 'dfk_mv'@'<host>';
```

Alle anderen Spalten (E-Mail, Status, Warteliste, Token, Zusatzwerte) bleiben für die MV nur über die View lesbar. Rückgeschriebene Änderungen lösen **keine** Mail aus. `updated_at` ändert sich dabei mit; die MV sieht ihre eigene Änderung beim nächsten Abgleich wieder – das ist unschädlich.

**`wp_dfk_mv_zusatzangaben_v1`** – Zusatzangaben als Zeilen, damit die MV kein JSON zerlegen muss (über `JSON_TABLE`, MySQL 8 / MariaDB 10.6)

| Spalte | Herkunft |
|---|---|
| `AnmeldungID` | `wp_dfk_registrations.id` |
| `Feld` | `key` aus der Felddefinition |
| `Bezeichnung` | `label` aus der Felddefinition |
| `Wert` | Wert aus `extra_values` |

Sensible Felder sind in dieser View nicht enthalten.

## 5. Schema-Entscheidungen (S1–S5 freigegeben 18.11.2026)

| ID | Entscheidung | Begründung |
|---|---|---|
| S1 | Status und Typen als `VARCHAR` + PHP-Enum | Neue Werte ohne Schemaänderung |
| S2 | Zeiten nur in deutscher Ortszeit | Nur Deutschland; MV arbeitet ebenso |
| S3 | Belegte Plätze per `COUNT`, kein Zähler | Geringe Datenmenge, keine Abweichungen möglich |
| S4 | IDs der Anmeldungen starten bei 1.000.000 | Schadet nicht und schließt Kollisionen mit alten TEC-IDs in der MV sicher aus |
| S5 | Bereinigung **30 Tage nach Veranstaltungsende**: alle Anmeldungen der Veranstaltung werden gelöscht, `purged_at` wird gesetzt ✅ | ADR 0005, entschieden 18.11.2026 |

## 6. Bewusst weggelassen

- Eigene Tabellen für Buchungen, Zusatzfelder, Feldwerte und Warteliste
- Migrationsspalten, Land, Zeitzonen, Geo-Index
- Double-Opt-in, Anmeldestart, Herkunft der Buchung
- Check-in (kommt bei Bedarf als eigene Tabelle dazu)
