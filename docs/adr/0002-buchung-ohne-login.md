# ADR 0002 – Buchung immer ohne Login möglich

Datum: 17.11.2026 · Status: **Angenommen** (Captain)

## Kontext

Laut AGENTS.md §7 können Teilnehmer DFK-Mitglieder oder Gäste sein, und für authentifizierte Mitglieder sollen bestehende WordPress-Konten verwendet werden. Offen war, ob für eine Buchung ein Login nötig ist.

## Entscheidung

Jede Buchung ist ohne Login möglich. Ein Login ist nie Voraussetzung. Ist jemand eingeloggt, werden lediglich die bekannten Daten (Name, E-Mail usw.) im Formular vorbelegt und die Buchung wird dem Konto zugeordnet.

## Folgen

- Die Buchung ohne Login ist der Normalfall und kein Sonderweg. Services, Validierung und Tests gehen von `user_id = 0` als regulärem Fall aus.
- Das Anlegen einer Buchung über die REST-API ist ein öffentlicher Endpunkt. Er braucht Missbrauchsschutz (Rate-Limit, Honeypot, ggf. Bestätigung per E-Mail) – siehe E-F17.
- ~~Verwaltungslink mit Token (E-F18)~~ – entfällt in V1: Stornos laufen ausschließlich über den Mitgliederservice (Entscheidung 18.11.2026).
- Mehrfachanmeldungen werden über die E-Mail-Adresse erkannt, bei Login zusätzlich über die User-ID – siehe E-F19.
- Die Teilnehmerart „Mitglied“ kann ohne Login nicht durch das Konto belegt werden. Wie sie festgestellt wird, ist noch offen – siehe E-F4.
- Die E-Mail-Adresse der anmeldenden Person ist Pflicht. Sie ist der einzige Kommunikationsweg.
