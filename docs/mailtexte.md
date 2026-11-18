# DFK Events – Mailtexte (Entwurf)

Stand: 18.11.2026 · Status: **Entwurf, wartet auf Freigabe** (O8)

Reine Textmails an die anmeldende Person. Gespeichert in `wp_dfk_settings` (ADR 0010), pflegbar über das CRM.

## Platzhalter

| Platzhalter | Inhalt |
|---|---|
| `{vorname}`, `{nachname}` | anmeldende Person |
| `{veranstaltung}` | Titel |
| `{datum}` | z. B. „Mittwoch, 18. November 2026“ |
| `{uhrzeit}` | z. B. „14:20 – 16:00 Uhr“ |
| `{ort}` | Name und Adresse des Orts bzw. „Online-Veranstaltung“ |
| `{veranstalter}` | Name des Veranstalters |
| `{link}` | Adresse der Veranstaltungsseite |
| `{teilnehmerliste}` | alle Personen der Buchung, je Zeile „Vorname Nachname (Mitglied/Nichtmitglied)“, bei Änderungen mit Status |
| `{kontakt}` | Kontaktadresse (Veranstalter-E-Mail bzw. zentrale Adresse) |

## 1. Buchung bestätigt

**Betreff:** Ihre Anmeldung: {veranstaltung} am {datum}

```
Guten Tag {vorname} {nachname},

vielen Dank für Ihre Anmeldung. Wir freuen uns, Sie bei folgender Veranstaltung begrüßen zu dürfen:

{veranstaltung}
{datum}, {uhrzeit}
{ort}
Veranstalter: {veranstalter}

Angemeldete Personen:
{teilnehmerliste}

Alle Informationen zur Veranstaltung: {link}

Falls Sie oder eine der angemeldeten Personen nicht teilnehmen können, geben Sie uns bitte kurz Bescheid: {kontakt}

Mit freundlichen Grüßen
Ihr DFK – Verband für Fach- und Führungskräfte e.V.
```

## 2. Buchung auf der Warteliste

**Betreff:** Warteliste: {veranstaltung} am {datum}

```
Guten Tag {vorname} {nachname},

vielen Dank für Ihr Interesse. Die Veranstaltung ist derzeit ausgebucht. Wir haben Ihre Anmeldung daher auf die Warteliste gesetzt:

{veranstaltung}
{datum}, {uhrzeit}
{ort}

Personen auf der Warteliste:
{teilnehmerliste}

Sobald ein Platz frei wird, melden wir uns bei Ihnen. Ihre Anmeldung ist erst mit dieser Nachricht verbindlich.

Mit freundlichen Grüßen
Ihr DFK – Verband für Fach- und Führungskräfte e.V.
```

## 3. Storno

**Betreff:** Stornierung: {veranstaltung} am {datum}

```
Guten Tag {vorname} {nachname},

wir bestätigen die Stornierung Ihrer Anmeldung bzw. einzelner Personen für:

{veranstaltung}
{datum}, {uhrzeit}

Aktueller Stand Ihrer Buchung:
{teilnehmerliste}

Bei Fragen erreichen Sie uns unter {kontakt}.

Mit freundlichen Grüßen
Ihr DFK – Verband für Fach- und Führungskräfte e.V.
```

## 4. Änderung (inkl. Nachrücken von der Warteliste)

**Betreff:** Änderung Ihrer Anmeldung: {veranstaltung} am {datum}

```
Guten Tag {vorname} {nachname},

an Ihrer Anmeldung hat sich etwas geändert:

{veranstaltung}
{datum}, {uhrzeit}
{ort}

Aktueller Stand Ihrer Buchung:
{teilnehmerliste}

Alle Informationen zur Veranstaltung: {link}

Bei Fragen erreichen Sie uns unter {kontakt}.

Mit freundlichen Grüßen
Ihr DFK – Verband für Fach- und Führungskräfte e.V.
```

## Hinweise

- „Änderung“ deckt auch das Nachrücken ab; die Teilnehmerliste zeigt dann den neuen Status („bestätigt“ statt „Warteliste“).
- Ändert sich die Veranstaltung selbst (Absage, Verschiebung), gehen **keine** automatischen Mails raus – das bleibt manuell, wie heute (ADR 0008).
- Anrede bewusst neutral („Guten Tag Vorname Nachname“), weil keine Anrede abgefragt wird.
