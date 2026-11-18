# DFK Events – Datenbankschema (Entwurf)

Stand: 18.11.2026 · Status: **Entwurf, wartet auf Freigabe durch den Captain**

Grundlage: [architektur.md](architektur.md), ADR 0001–0008, [mv-schnittstelle.md](mv-schnittstelle.md).

**Leitlinie:** 100–120 Veranstaltungen pro Jahr, nur Deutschland, keine Migration (ADR 0006). **Zwei Tabellen**, Aufbau gleich wie künftig in der MV (ADR 0007).

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
   │ 1:n
   ▼
wp_dfk_registrations                  eine Zeile je Person          ≙ MV T_Veranstaltungen_Anmeldungen
                                      zusammengefasst über booking_id

wp_dfk_mv_sync                        Sync-Bestätigungen der MV
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
| `venue_name` | VARCHAR(255) NULL | **Ort direkt am Termin** (ADR 0007) |
| `venue_street` | VARCHAR(255) NULL | |
| `venue_postal_code` | VARCHAR(5) NULL | |
| `venue_city` | VARCHAR(255) NULL | |
| `venue_lat` | DECIMAL(9,6) NULL | beim Speichern ermittelt, korrigierbar |
| `venue_lng` | DECIMAL(9,6) NULL | |
| `directions` | TEXT NULL | Anfahrtsinformationen |
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

Index: `(starts_at)`. Der Veranstalter hängt als Taxonomie-Term am Beitrag (3.5).

### 3.2 `wp_dfk_registrations` – Anmeldungen (eine Zeile je Person)

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
| `token_hash` | CHAR(64) NULL | Verwaltungslink, nur in der Anmelderzeile |
| `created_at`, `updated_at` | DATETIME NOT NULL | `created_at` = Reihenfolge der Warteliste |

Indizes: `(event_id, status, waitlist)`, `(booking_id)`, `(event_id, email)`, `(updated_at)`, `UNIQUE (token_hash)`.

**Belegte Plätze** = Anzahl Zeilen mit `status = 'active'` und `waitlist = 0`.

**Die anmeldende Person kommt nicht selbst:** Ihre Zeile bleibt als Anmelderzeile (Kontakt, Token) bestehen, mit `status = 'not_attending'`; sie zählt nicht als Platz.

**Neue Buchung:** In einer Transaktion wird die Veranstaltung gesperrt (`SELECT … FOR UPDATE`), die Anmelderzeile eingefügt, ihre `id` als `booking_id` gesetzt und die Gäste mit derselben `booking_id` eingefügt.

### 3.3 Zusatzfelder als JSON

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

### 3.4 `wp_dfk_mv_sync`

| Spalte | Typ | Bemerkung |
|---|---|---|
| `id` | BIGINT UNSIGNED **PK** AUTO_INCREMENT | |
| `synced_at` | DATETIME NOT NULL | Zeitpunkt des erfolgreichen MV-Laufs |
| `records` | INT UNSIGNED NULL | Anzahl übernommener Anmeldungen |

Die MV darf hier nur `INSERT`. Bereinigt werden Veranstaltungen, deren Ende vor dem jüngsten `synced_at` liegt, spätestens 30 Tage nach Ende (ADR 0005): alle Anmeldungen der Veranstaltung werden gelöscht, `purged_at` wird gesetzt.

### 3.5 Veranstalter – Taxonomie `dfk_unit`

Jede Veranstaltung hat genau einen Veranstalter aus der MV-Liste `T_Veranstalter` (einmaliger CSV-Import, danach Pflege im WP-Admin).

| Term-Meta | Bemerkung |
|---|---|
| `mv_veranstalter_id` | = `T_Veranstalter.InterneID`, Pflicht, eindeutig |
| `unit_type` | `dfk`, `regionalgruppe`, `netzwerk`, `kooperation` |
| `active` | inaktive Veranstalter sind für neue Veranstaltungen nicht wählbar |

Reine Online-Veranstaltungen gehören immer dem DFK; hybride dem Veranstalter vor Ort.

## 4. Views für die MV

Die MV liest nur diese beiden Views. Sie bilden die künftigen MV-Tabellen `T_Veranstaltungen` und `T_Veranstaltungen_Anmeldungen` nahezu 1:1 ab.

**`wp_dfk_mv_veranstaltungen_v1`**

| Spalte | Herkunft |
|---|---|
| `VeranstaltungID` | `post_id` |
| `Titel` | `wp_posts.post_title` |
| `VeranstalterInterneID`, `Veranstalter` | Term-Meta / Term-Name |
| `Startdatum`, `Enddatum` | `starts_at`, `ends_at` |
| `Anmeldeschluss` | `registration_closes_at` |
| `Format` | `format` |
| `Ort`, `PLZ`, `Stadt` | `venue_name`, `venue_postal_code`, `venue_city` |
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
| `TeilnehmerartAngabe` | `Mitglied` / `Nichtmitglied` |
| `Status` | `aktiv`, `storniert`, `nimmt_nicht_teil` |
| `Warteliste` | `waitlist` |
| `Zusatzangaben` | `extra_values` **ohne** sensible Felder |
| `Anmeldedatum` | `created_at` |
| `LastUpdate` | `updated_at` |

## 5. Schema-Entscheidungen zur Freigabe

| ID | Entscheidung | Begründung |
|---|---|---|
| S1 | Status und Typen als `VARCHAR` + PHP-Enum | Neue Werte ohne Schemaänderung |
| S2 | Zeiten nur in deutscher Ortszeit | Nur Deutschland; MV arbeitet ebenso |
| S3 | Belegte Plätze per `COUNT`, kein Zähler | Geringe Datenmenge, keine Abweichungen möglich |
| S4 | IDs der Anmeldungen starten oberhalb der alten TEC-Attendee-IDs (z. B. 1.000.000) | Nur nötig, wenn die alten Anmeldungen in der MV-Tabelle mit eindeutiger ID bleiben; entfällt, wenn die MV mit `T_Veranstaltungen_Anmeldungen` neu beginnt |
| S5 | Bereinigung löscht alle Anmeldungen einer Veranstaltung, `purged_at` wird gesetzt | ADR 0005 |

## 6. Bewusst weggelassen

- Eigene Tabellen für Buchungen, Orte, Zusatzfelder, Feldwerte und Warteliste
- Migrationsspalten, Land, Zeitzonen, Geo-Index
- Double-Opt-in, Anmeldestart, Herkunft der Buchung
- Check-in (kommt bei Bedarf als eigene Tabelle dazu)
