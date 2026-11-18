# ADR 0008 – Warteliste manuell, Mails nur bei Buchung, Storno, Änderung

Datum: 18.11.2026 · Status: **Angenommen** (Captain)

## Entscheidung

**Warteliste**
- Ist eine Veranstaltung voll, landen neue Anmeldungen mit gesetztem Wartelisten-Flag in der Anmeldetabelle.
- **Kein automatisches Nachrücken.** Der Mitgliederservice entscheidet, wer nachrückt.
- Die Warteliste einer Veranstaltung ist im WordPress-Backend und in der MV sichtbar, in der Reihenfolge der Anmeldung.
- Keine Reservierungsfristen.

**Automatische Mails** gehen nur an die anmeldende Person, und nur bei:
- Buchung (Bestätigung bzw. „auf der Warteliste“, mit Liste aller angemeldeten Personen),
- Stornierung,
- Änderung (dazu zählt auch das Nachrücken von der Warteliste).

Keine Erinnerungsmails, keine Mails an einzelne Gäste.

## Vorschlag zur Freigabe: Gruppenbuchung bei knappen Plätzen

Passt eine Buchung nicht mehr vollständig in die freien Plätze, kommt **die ganze Buchung** auf die Warteliste. Der Mitgliederservice kann einzelne Personen daraus nachrücken lassen. So wird keine Gruppe ungefragt getrennt.

## Folgen

- Kein `WaitlistService` mit eigener Logik; Buchung setzt das Flag, das Backend erlaubt das Zurücksetzen (= Nachrücken).
- Kapazität: Personen mit Wartelisten-Flag zählen nicht als belegte Plätze.
- Nachrücken ist eine Änderung und löst die Änderungsmail aus.
