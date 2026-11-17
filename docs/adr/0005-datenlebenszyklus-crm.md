# ADR 0005 – Datenlebenszyklus: Übernahme ins CRM, danach Löschung

Datum: 17.11.2026 · Status: **Angenommen** (Captain), Details offen

## Kontext

Das DFK-CRM übernimmt die Anmelde- und Teilnehmerdaten aus DFK Events. DFK Events ist damit **kein dauerhafter Datenspeicher** für personenbezogene Daten, sondern ein Zwischenspeicher bis zur Übernahme ins CRM.

## Entscheidung

1. Das CRM übernimmt die Daten **normalerweise direkt nach dem Veranstaltungsdatum**.
2. Nach erfolgreicher Übernahme werden die personenbezogenen Daten in DFK Events **gelöscht**.
3. **Spätestens 30 Tage nach Veranstaltungsende** wird automatisch gelöscht – auch wenn keine Übernahme stattgefunden hat (Sicherheitsnetz). Die Frist ist als Einstellung änderbar.
4. Für sensible Zusatzfelder (ADR/E-F24) gilt dieselbe Regel.

5. Das CRM ist die **Mitgliederverwaltungs-Software (MV-Software)** des DFK. Sie übernimmt die Daten und erstellt **alle Auswertungen** (z. B. nach Regionalgruppe, Mitglied/Nichtmitglied). DFK Events braucht daher **keine eigenen Statistik- oder Auswertungsfunktionen**.

## Vorschlag zur Umsetzung (zur Freigabe)

- Jede Buchung erhält einen Übertragungsstatus: *offen* → *an CRM übertragen* (mit Zeitpunkt) → *bereinigt*.
- Löschung erfolgt per WP-Cron durch den `PrivacyService`.
- **Bereinigen heißt: Buchungen, Teilnehmer und Zusatzfeldwerte vollständig löschen.** Da die MV-Software alle Auswertungen übernimmt, muss DFK Events keine anonymisierten Reste aufbewahren. Die Veranstaltung selbst (Beitrag, Termin, Ort) bleibt erhalten.
- Der Export an die MV-Software muss alles enthalten, was dort ausgewertet wird – mindestens Veranstaltung, Datum, Regionalgruppe, Buchung, alle Teilnehmer mit Teilnehmerart und Status sowie die Zusatzfelder.
- Admins sehen vor der Löschfrist, welche Events noch nicht übertragen wurden.

## Offene Punkte

- **E-F25:** Welches Produkt ist die MV-Software, und welche Import-Schnittstellen bietet sie?
- **E-F26:** Wie übernimmt die MV-Software die Daten? (holt per REST-API ab / CSV-Import / direkter Datenbankzugriff) – hängt von E-F25 ab.
- **E-F27:** Wer löst die Löschung aus: Bestätigung durch die MV-Software nach erfolgreicher Übernahme oder ausschließlich die Frist?
- ~~E-F28: Umfang der Löschung~~ – **geklärt:** alle personenbezogenen Buchungsdaten werden gelöscht, keine Statistik in DFK Events.

## Folgen

- „Meine Buchungen“ zeigt nur aktuelle und gerade vergangene Veranstaltungen; nach der Bereinigung verschwinden die Buchungen dort.
- Die Verwaltungslinks (E-F18) werden mit der Bereinigung ungültig.
- Für den späteren Check-in über die App bleiben die Daten bis zum Ende der Veranstaltung vollständig erhalten.
- Löschfristen (E-F15) sind damit weitgehend beantwortet.
