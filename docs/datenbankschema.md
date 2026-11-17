# DFK Events – Datenbankschema (Entwurf)

Stand: 17.11.2026 · Status: **Entwurf, wartet auf Freigabe durch den Captain**

Grundlage: [architektur.md](architektur.md), ADR 0001–0005, [mv-schnittstelle.md](mv-schnittstelle.md).

---

## 1. Allgemeine Regeln

- MySQL 8.0 / MariaDB 10.6, **InnoDB**, Zeichensatz aus `$wpdb->get_charset_collate()` (utf8mb4).
- Alle Tabellennamen mit WordPress-Präfix, im Folgenden `wp_`.
- Status- und Typwerte als `VARCHAR` statt `ENUM` – neue Werte brauchen dann keine Schemaänderung. Erlaubte Werte legt jeweils ein PHP-Enum fest.
- Keine Fremdschlüssel-Constraints auf WordPress-Tabellen (WordPress nutzt selbst keine). Innerhalb der Plugin-Tabellen: Fremdschlüssel ja, damit Bereinigung per `ON DELETE CASCADE` vollständig ist.
- Zeitstempel `created_at` / `updated_at` in jeder Tabelle. `updated_at` ist für den inkrementellen MV-Abgleich wichtig.
- Schemaänderungen nur über versionierte Migrationen (`Database/Migrations`), Stand in Option `dfk_events_db_version`.

## 2. Übersicht

```
wp_posts (post_type = dfk_event)
   │ 1:1
   ▼
wp_dfk_events ──────── n:1 ──▶ wp_dfk_event_locations
   │ 1:n
   ├──▶ wp_dfk_event_fields
   │ 1:n
   ▼
wp_dfk_event_bookings          (anmeldende Person)
   │ 1:n
   ▼
wp_dfk_event_attendees         (Teilnehmer, eigener Status)

wp_dfk_event_field_values ── gehört zu Feld + Buchung (+ Teilnehmer)
wp_dfk_mv_sync                 (Sync-Bestätigungen der MV)
```

## 3. Tabellen

### 3.1 `wp_dfk_events` – strukturierte Eventdaten

Titel, Beschreibung, Bild, Veröffentlichungsstatus und Permalink liegen im Beitrag (`wp_posts`).

| Spalte | Typ | Bemerkung |
|---|---|---|
| `post_id` | BIGINT UNSIGNED **PK** | = `wp_posts.ID` (1:1, E-T5) |
| `starts_at` | DATETIME NOT NULL | UTC |
| `ends_at` | DATETIME NOT NULL | UTC |
| `starts_at_local` | DATETIME NOT NULL | Ortszeit Europe/Berlin, für Anzeige und MV (siehe S2) |
| `ends_at_local` | DATETIME NOT NULL | dito |
| `all_day` | TINYINT(1) NOT NULL DEFAULT 0 | |
| `event_status` | VARCHAR(20) NOT NULL DEFAULT 'scheduled' | `scheduled`, `cancelled`, `postponed` (E-F6, vorläufig) |
| `format` | VARCHAR(10) NOT NULL DEFAULT 'onsite' | `onsite`, `online`, `hybrid` |
| `location_id` | BIGINT UNSIGNED NULL | FK → locations; NULL bei reinen Online-Events |
| `online_url` | VARCHAR(2048) NULL | nur für bestätigte Teilnehmer sichtbar (E-F13) |
| `organizer` | VARCHAR(255) NULL | **vorläufig Freitext** – E-F7 offen |
| `registration_enabled` | TINYINT(1) NOT NULL DEFAULT 1 | |
| `registration_opens_at` | DATETIME NULL | UTC, NULL = sofort |
| `registration_closes_at` | DATETIME NULL | UTC, Anmeldeschluss |
| `capacity` | INT UNSIGNED NULL | NULL = unbegrenzt |
| `seats_taken` | INT UNSIGNED NOT NULL DEFAULT 0 | Zähler bestätigter Teilnehmer, nur im gesperrten Buchungsvorgang geändert (S3) |
| `max_per_booking` | SMALLINT UNSIGNED NOT NULL DEFAULT 5 | inkl. anmeldender Person, falls sie teilnimmt |
| `allow_members` | TINYINT(1) NOT NULL DEFAULT 1 | zulässige Teilnehmerarten (ADR 0003) |
| `allow_non_members` | TINYINT(1) NOT NULL DEFAULT 1 | |
| `allow_multiple_bookings` | TINYINT(1) NOT NULL DEFAULT 0 | mehrere Buchungen je E-Mail (E-F19) |
| `waitlist_enabled` | TINYINT(1) NOT NULL DEFAULT 1 | Regeln E-F9 offen |
| `require_confirmation` | TINYINT(1) NOT NULL DEFAULT 0 | Double-Opt-in (E-F17) |
| `purged_at` | DATETIME NULL | Zeitpunkt der Bereinigung (ADR 0005) |
| `created_at`, `updated_at` | DATETIME NOT NULL | |

Indizes: `(starts_at)`, `(event_status, starts_at)`, `(location_id)`, `(updated_at)`.

Die **Regionalgruppe** (E-F5) ist hier bewusst nicht enthalten – Vorschlag Taxonomie `dfk_region` in den WordPress-Tabellen.

### 3.2 `wp_dfk_event_locations` – Veranstaltungsorte

| Spalte | Typ | Bemerkung |
|---|---|---|
| `id` | BIGINT UNSIGNED **PK** AUTO_INCREMENT | |
| `name` | VARCHAR(255) NOT NULL | |
| `street` | VARCHAR(255) NULL | Straße und Hausnummer |
| `postal_code` | VARCHAR(10) NULL | |
| `city` | VARCHAR(255) NULL | |
| `country` | CHAR(2) NOT NULL DEFAULT 'DE' | ISO 3166-1 |
| `latitude` | DECIMAL(9,6) NULL | |
| `longitude` | DECIMAL(9,6) NULL | |
| `geocode_source` | VARCHAR(20) NULL | `nominatim`, `manual` |
| `geocoded_at` | DATETIME NULL | |
| `directions` | TEXT NULL | Anfahrtsinformationen |
| `is_active` | TINYINT(1) NOT NULL DEFAULT 1 | statt Löschen, solange Events darauf verweisen |
| `legacy_tec_venue_id` | BIGINT UNSIGNED NULL | Migration |
| `created_at`, `updated_at` | DATETIME NOT NULL | |

Indizes: `(latitude, longitude)` für den Bounding-Box-Vorfilter (E-T11), `(postal_code)`.

### 3.3 `wp_dfk_event_bookings` – Buchungen

| Spalte | Typ | Bemerkung |
|---|---|---|
| `id` | BIGINT UNSIGNED **PK** AUTO_INCREMENT | |
| `event_id` | BIGINT UNSIGNED NOT NULL | FK → events.post_id, CASCADE |
| `user_id` | BIGINT UNSIGNED NULL | WP-Benutzer, NULL bei Buchung ohne Login (ADR 0002) |
| `first_name` | VARCHAR(100) NOT NULL | anmeldende Person |
| `last_name` | VARCHAR(100) NOT NULL | |
| `email` | VARCHAR(254) NOT NULL | |
| `email_normalized` | VARCHAR(254) NOT NULL | klein geschrieben, getrimmt – für Mehrfachanmeldung (E-F19) |
| `status` | VARCHAR(20) NOT NULL | `pending`, `confirmed`, `cancelled` |
| `source` | VARCHAR(10) NOT NULL DEFAULT 'web' | `web`, `app`, `admin`, `import` |
| `access_token_hash` | CHAR(64) NULL | SHA-256 des Verwaltungslinks (E-F18); Klartext nur in der Mail |
| `confirm_token_hash` | CHAR(64) NULL | nur bei Double-Opt-in |
| `confirm_expires_at` | DATETIME NULL | |
| `legacy_tec_order_id` | VARCHAR(64) NULL | Migration (`_tribe_rsvp_order`) |
| `created_at`, `updated_at` | DATETIME NOT NULL | |
| `cancelled_at` | DATETIME NULL | |

Indizes: `(event_id, email_normalized)`, `(user_id)`, `UNIQUE (access_token_hash)`, `(updated_at)`.

### 3.4 `wp_dfk_event_attendees` – Teilnehmer

| Spalte | Typ | Bemerkung |
|---|---|---|
| `id` | BIGINT UNSIGNED **PK** AUTO_INCREMENT | Startwert siehe S4 |
| `booking_id` | BIGINT UNSIGNED NOT NULL | FK → bookings, CASCADE |
| `event_id` | BIGINT UNSIGNED NOT NULL | redundant zur Buchung – für schnelles Zählen |
| `is_booker` | TINYINT(1) NOT NULL DEFAULT 0 | anmeldende Person nimmt teil (ADR 0004) → MV `GastStatus` H/B |
| `first_name` | VARCHAR(100) NOT NULL | |
| `last_name` | VARCHAR(100) NOT NULL | |
| `email` | VARCHAR(254) NOT NULL | Pflicht (ADR 0004), darf innerhalb einer Buchung mehrfach vorkommen (E-F22) |
| `attendee_type` | VARCHAR(20) NOT NULL | `member`, `non_member` (ADR 0003) |
| `status` | VARCHAR(20) NOT NULL | `pending`, `confirmed`, `waitlisted`, `cancelled` (später `checked_in`) |
| `waitlisted_at` | DATETIME NULL | Reihenfolge der Warteliste |
| `legacy_tec_attendee_id` | BIGINT UNSIGNED NULL | Migration |
| `created_at`, `updated_at` | DATETIME NOT NULL | |
| `cancelled_at` | DATETIME NULL | |

Indizes: `(event_id, status)`, `(booking_id)`, `(event_id, waitlisted_at)`, `(updated_at)`.

### 3.5 `wp_dfk_event_fields` – Definition der Zusatzfelder

| Spalte | Typ | Bemerkung |
|---|---|---|
| `id` | BIGINT UNSIGNED **PK** AUTO_INCREMENT | |
| `event_id` | BIGINT UNSIGNED NOT NULL | FK → events, CASCADE |
| `field_key` | VARCHAR(64) NOT NULL | technischer Name, eindeutig je Event |
| `label` | VARCHAR(255) NOT NULL | |
| `help_text` | VARCHAR(500) NULL | |
| `field_type` | VARCHAR(20) NOT NULL | `text`, `textarea`, `select`, `checkbox`, `date` |
| `scope` | VARCHAR(10) NOT NULL | `booking`, `attendee` |
| `is_required` | TINYINT(1) NOT NULL DEFAULT 0 | |
| `is_sensitive` | TINYINT(1) NOT NULL DEFAULT 0 | E-F24: verschlüsselt, eingeschränkt sichtbar |
| `options` | JSON NULL | Auswahlwerte bei `select` |
| `sort_order` | SMALLINT NOT NULL DEFAULT 0 | |
| `created_at`, `updated_at` | DATETIME NOT NULL | |

Index: `UNIQUE (event_id, field_key)`.

### 3.6 `wp_dfk_event_field_values` – Werte der Zusatzfelder

| Spalte | Typ | Bemerkung |
|---|---|---|
| `id` | BIGINT UNSIGNED **PK** AUTO_INCREMENT | |
| `field_id` | BIGINT UNSIGNED NOT NULL | FK → fields, CASCADE |
| `booking_id` | BIGINT UNSIGNED NOT NULL | FK → bookings, CASCADE |
| `attendee_id` | BIGINT UNSIGNED NOT NULL DEFAULT 0 | 0 bei Feldern mit `scope = booking` (statt NULL, damit der UNIQUE-Index greift) |
| `value` | TEXT NULL | bei sensiblen Feldern verschlüsselt (Schlüssel in `wp-config.php`) |
| `is_encrypted` | TINYINT(1) NOT NULL DEFAULT 0 | |
| `created_at`, `updated_at` | DATETIME NOT NULL | |

Index: `UNIQUE (field_id, booking_id, attendee_id)`.

### 3.7 `wp_dfk_mv_sync` – Sync-Bestätigung der MV

| Spalte | Typ | Bemerkung |
|---|---|---|
| `id` | BIGINT UNSIGNED **PK** AUTO_INCREMENT | |
| `synced_at` | DATETIME NOT NULL | Zeitpunkt des erfolgreichen MV-Laufs (UTC) |
| `client` | VARCHAR(50) NOT NULL | z. B. Rechnername der MV |
| `records` | INT UNSIGNED NULL | Anzahl übernommener Teilnehmer, zur Kontrolle |

Die MV erhält auf diese Tabelle **nur** `INSERT`. Bereinigt werden Events, deren Ende vor dem jüngsten `synced_at` liegt (ADR 0005).

### 3.8 Später: `wp_dfk_event_checkins`

Für Check-in/QR-Code vorgesehen, **nicht Teil von V1**. Skizze: `attendee_id`, `checked_in_at`, `checked_in_by`, `device`, `code_hash`.

## 4. Views für die MV (Schnittstelle v1)

Die MV liest ausschließlich diese Views. Spaltennamen orientieren sich an `T_Termine` / `T_Termine_Anmeldungen`, damit die Anpassung in VBA klein bleibt.

**`wp_dfk_mv_events_v1`**

| Spalte | Herkunft |
|---|---|
| `TerminID` | `events.post_id` |
| `Titel` | `wp_posts.post_title` |
| `Veranstalter` | `events.organizer` |
| `Startdatum`, `Enddatum` | `starts_at_local`, `ends_at_local` |
| `Anmeldeschluss` | `registration_closes_at` in Ortszeit |
| `Ort`, `OrtLocation` | `locations.name`, `locations.city` |
| `Abgesagt` | `event_status = 'cancelled'` |
| `MaxTeilnehmer` | `capacity` |
| `Veroeffentlicht` | `wp_posts.post_status = 'publish'` |
| `LastUpdate` | jüngstes `updated_at` aus Event und Beitrag |

**`wp_dfk_mv_attendees_v1`**

| Spalte | Herkunft |
|---|---|
| `Termin_AnmeldungID` | `attendees.id` |
| `TerminID` | `attendees.event_id` |
| `HauptbuchungsID` | `bookings.id` |
| `GastStatus` | `H` wenn `is_booker = 1`, sonst `B` |
| `Vorname`, `Nachname`, `EMail` | Teilnehmer |
| `AnmelderVorname`, `AnmelderNachname`, `AnmelderEMail` | Buchung (auch wenn die anmeldende Person nicht teilnimmt) |
| `TeilnehmerartAngabe` | `Mitglied` / `Nichtmitglied` – Selbstauskunft, die MV ermittelt den Status weiterhin selbst |
| `Status` | `bestaetigt`, `warteliste`, `storniert`, `offen` |
| `Anmeldedatum` | `attendees.created_at` (Ortszeit) |
| `LastUpdate` | `attendees.updated_at` |

Zusatzfelder werden als eigene View `wp_dfk_mv_field_values_v1` (Teilnehmer-ID, Feldname, Wert) bereitgestellt – **ohne** sensible Felder, sofern nicht ausdrücklich freigegeben.

## 5. Schema-Entscheidungen zur Freigabe

| ID | Entscheidung | Begründung |
|---|---|---|
| S1 | Status/Typen als `VARCHAR` + PHP-Enum statt `ENUM` | Erweiterbar ohne Migration |
| S2 | Zeiten **doppelt**: UTC für Logik, Ortszeit für Anzeige und MV | MV und Access brauchen Ortszeit; `CONVERT_TZ` mit Zeitzonennamen setzt geladene Zeitzonentabellen in MySQL voraus, die beim Hoster nicht garantiert sind |
| S3 | Zähler `seats_taken` zusätzlich zu den Teilnehmerzeilen | Übersicht mit vielen Events ohne `COUNT` je Event; geändert wird er nur innerhalb der Sperre des Buchungsvorgangs. Ein Admin-Werkzeug kann ihn aus den Teilnehmern neu berechnen |
| S4 | `AUTO_INCREMENT` der Teilnehmer startet **oberhalb der höchsten TEC-Attendee-ID** | Die MV hat einen eindeutigen Schlüssel auf `Termin_AnmeldungId`, der heute TEC-Post-IDs enthält. Ohne Abstand könnten neue IDs mit alten kollidieren |
| S5 | Bei der Migration **behalten Events ihre Post-ID** (Umwandlung `tribe_events` → `dfk_event` statt Neuanlage) | Die `TerminID` in der MV bleibt gültig, alte Anmeldungen in der MV bleiben dem richtigen Termin zugeordnet; URLs lassen sich leichter erhalten |
| S6 | Bereinigung = `DELETE` der Buchungen (Teilnehmer und Feldwerte folgen per CASCADE) und `purged_at` am Event | ADR 0005: keine Statistik im Plugin |

## 6. Noch offen und schema-relevant

- **E-F5 Regionalgruppe** – Taxonomie (Vorschlag) oder eigene Tabelle? Sollen die Kennungen denen der MV entsprechen?
- **E-F7 Veranstalter** – bleibt es beim Freitext?
- **E-F6 Eventstatus** – reichen `scheduled`, `cancelled`, `postponed`?
- **E-F9 Warteliste** – das Schema unterstützt Status `waitlisted` und Reihenfolge; die Regeln selbst sind offen.
- **E-F2 Wiederkehrende Events** – nicht abgebildet; „Event duplizieren“ braucht keine Schemaänderung.
