# ADR 0004 – Anmeldende Person und Teilnehmer

Datum: 17.11.2026 · Status: **Angenommen** (Captain)

## Kontext

AGENTS.md §7 trennt die anmeldende Person und die tatsächlichen Teilnehmer. Zu klären war, ob die anmeldende Person immer selbst teilnimmt.

## Entscheidung

- Eine **Buchung** hat genau eine **anmeldende Person** (Kontaktdaten, E-Mail für Bestätigung und Verwaltungslink).
- Eine Buchung enthält einen oder mehrere **Teilnehmer**.
- Die anmeldende Person kann selbst Teilnehmer sein. Im Formular gibt es dafür das Häkchen **„Ich nehme selbst teil“**, das standardmäßig gesetzt ist. Ohne Häkchen meldet sie nur andere Personen an.
- Jeder Teilnehmer hat eine eigene ID, einen eigenen Status und eine eigene Teilnehmerart Mitglied/Nichtmitglied (ADR 0003).
- **Von jedem Teilnehmer werden eigene Personendaten erfasst**, auch von Gästen. Ein Gast ist also nicht nur „+1“. Feste Pflichtfelder für **jeden** Teilnehmer:

  | Feld | Pflicht |
  |---|---|
  | Vorname | ja |
  | Nachname | ja |
  | Mitglied / Nichtmitglied (ADR 0003) | ja |
  | E-Mail | ja |

  Weitere Angaben (z. B. Personalausweisnummer) werden **pro Event optional** über die dynamischen Zusatzfelder ergänzt (AGENTS.md §8). Dort legt der Admin je Feld fest: Feldtyp, Pflicht oder freiwillig, für die Buchung oder für jeden Teilnehmer einzeln, sensibel ja/nein (E-F24).
- Nimmt die anmeldende Person selbst teil, werden ihre Daten nur einmal eingegeben und für den Teilnehmer übernommen. Bei Login werden sie aus dem Konto vorbelegt.

## Beispiel

Ein Mitglied meldet sich mit zwei weiteren Personen an:

| Teilnehmer | Rolle in der Buchung | Teilnehmerart |
|---|---|---|
| 1. Anmeldendes Mitglied | anmeldende Person, nimmt selbst teil | Mitglied |
| 2. Ehefrau | Gast (Begleitperson) | Mitglied |
| 3. Freund | Gast (Begleitperson) | Nichtmitglied |

Ergebnis: **1 Buchung, 3 Teilnehmer**. Die Kapazität des Events verringert sich um 3. Jeder der drei kann einzeln storniert werden.

## Folgen

- Im Datenmodell trägt der Teilnehmer ein Kennzeichen „ist anmeldende Person“. Die Rolle „Gast“ ergibt sich daraus und wird nicht separat gespeichert.
- Die maximale Personenzahl je Buchung (konfigurierbar pro Event) zählt alle Teilnehmer, einschließlich der anmeldenden Person, falls sie teilnimmt.
- Wird der Teilnehmer storniert, der zugleich anmeldende Person ist, bleibt die Buchung mit den übrigen Teilnehmern bestehen. Die anmeldende Person bleibt Kontakt der Buchung.
