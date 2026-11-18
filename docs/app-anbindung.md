# DFK Events – Anbindung der DFK-App

Stand: 18.11.2026 · Status: **3.1 freigegeben (mit Absicherung), 3.2 vorläufig – Prüfung am Prototyp**

Quelle: Repository `dfk-app` (Alpha). Gelesen wurden `api/events.php`, `api/lib/wordpress.php`, `api/login.php`, `src/distance.ts` und `toDo.md`; keine Konfigurations- oder Zugangsdateien.

## 1. Ist-Zustand der App

- React-Frontend plus **eigenes kleines PHP-Backend** (`api/`) auf eigenem Hosting (Mittwald).
- Termine holt das Backend **lesend** über die öffentliche REST-API von The Events Calendar (`/wp-json/tribe/events/v1/events`) und hält sie 5 Minuten im Dateicache.
- Die App nutzt davon: ID, Titel, Beschreibung, URL, ganztägig, Beginn/Ende, Kategorien, Ort (Name, Adresse, PLZ, Stadt), `ticketed`.
- Login der App über eine eigene Benutzertabelle, nicht über WordPress.
- Eine Terminanmeldung in der App gibt es noch nicht. Bewusste Entscheidung im Piloten: Die App verlinkt nicht auf dfk.eu, weil das am iPhone aus der App herausführt.

Behelfe in der App, die DFK Events überflüssig macht:

| Behelf heute | Mit DFK Events |
|---|---|
| Entfernung über fest eingetragene Stadtkoordinaten (`distance.ts`) | Koordinaten je Ort aus `wp_dfk_venues` |
| Online-Termine nur über Kategorie „Online“ bzw. Ort „Zoom“ erkennbar | Feld `format` (`online`, `onsite`, `hybrid`) |
| Anmeldeschluss nur im Beschreibungstext | Feld `registration_closes_at` |
| Kein Hinweis auf freie Plätze / Warteliste | Freie Plätze bzw. „Warteliste“ berechnet |

## 2. Folge der TEC-Ablösung

Mit der Deaktivierung von The Events Calendar **entfällt der Endpunkt, aus dem die App ihre Termine liest.** DFK Events muss einen Ersatz bieten, sonst zeigt die App keine Termine mehr.

## 3. Vorschlag

### 3.1 Termine lesen: ein öffentlicher, nur lesender Endpunkt

`GET /wp-json/dfk-events/v1/events` – öffentlich, nur lesend, ohne personenbezogene Daten. Liefert je kommender Veranstaltung:

`id`, `title`, `description`, `url`, `starts_at`, `ends_at`, `all_day`, `format`, `status`, `veranstalter`, `venue` (Name, Straße, PLZ, Stadt, `lat`, `lng`), `registration_open`, `registration_closes_at`, `seats_left` bzw. `waitlist`.

- Ersetzt den TEC-Endpunkt nahezu 1:1; im App-Backend ändert sich nur `api/events.php`.
- Kein Schreibzugriff über REST. Die Website selbst braucht für das Buchen keinen REST-Endpunkt (normaler Formularversand).
**Freigegeben (Captain, 18.11.2026) mit entsprechender Absicherung:**

| Maßnahme | Umsetzung |
|---|---|
| Nur lesen | Ausschließlich `GET`; keine weiteren Routen unter `dfk-events/v1` |
| Keine personenbezogenen Daten | Nur Veranstaltungsdaten; keine Anmeldungen, Namen, E-Mails; **kein Online-Zugangslink** |
| Nur Veröffentlichtes | Nur Beiträge mit Status `publish`, nur laufende und kommende Termine |
| Feste Antwortform | Feste Feldliste, keine frei wählbaren Felder oder Filter außer Seite/Zeitraum; Eingaben werden geprüft |
| Last begrenzen | Antwort wird serverseitig zwischengespeichert (5 Minuten); höchstens 100 Einträge je Seite; Anfragebegrenzung je IP |
| App-Kennung (optional) | Das App-Backend sendet einen geheimen Schlüssel im Header (`X-DFK-App-Key`, hinterlegt in `wp-config.php`). Ohne Schlüssel gilt eine strengere Anfragebegrenzung. |
| Abschaltbar | Endpunkt per Einstellung deaktivierbar |

Die Alternative ohne REST (App-Backend liest die Views direkt per MySQL) entfällt damit.

### 3.2 Buchen: über die WordPress-Seite, ohne die App sichtbar zu verlassen

Die App bucht über die Website (Entscheidung Captain). Damit das zur Pilot-Entscheidung „nicht aus der App herausführen“ passt:

- DFK Events stellt das Buchungsformular zusätzlich als **reduzierte Seite ohne Kopf, Fuß und Navigation** bereit, z. B. `/event/<slug>/anmelden/?ansicht=app`.
- Die App öffnet diese Seite in einer In-App-Ansicht (iOS: `SFSafariViewController`, Android: Custom Tab). Der Nutzer bleibt optisch in der App.
- Buchungslogik, Kapazitätsprüfung, Warteliste und Mails bleiben ausschließlich im Plugin.

**Status:** vorläufig angenommen. Der Captain entscheidet endgültig, wenn ein Prototyp der reduzierten Anmeldeseite zu sehen ist. Der Aufwand ist gering, weil es dasselbe Formular mit einem schlankeren Template ist.

### 3.3 „Meine Termine“ (später)

Über die von der MV zurückgeschriebene Mitgliedsnummer bzw. die E-Mail-Adresse (ADR 0009) lassen sich die Anmeldungen eines Mitglieds finden. Wie die App das abruft, wird entschieden, wenn die App-Anmeldung (bisher eigene Benutzertabelle, später ggf. OpenID Connect mit WordPress) feststeht.
