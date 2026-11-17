# ADR 0003 – Teilnehmerart per Selbstauskunft

Datum: 17.11.2026 · Status: **Angenommen** (Captain)

## Kontext

Teilnehmer können DFK-Mitglieder oder Gäste sein (AGENTS.md §7). Da Buchen immer ohne Login möglich ist (ADR 0002), kann die Mitgliedschaft nicht über das WordPress-Konto belegt werden.

## Entscheidung

Für **jede Person einer Buchung** – die anmeldende Person ebenso wie alle mit angemeldeten Gäste (Begleitpersonen) – enthält das Anmeldeformular ein **Dropdown** mit den Werten:

- **Mitglied**
- **Nichtmitglied**

Es ist eine Selbstauskunft und wird nicht geprüft. Eine Mitgliedsnummer wird **nicht** erfasst.

## Umsetzungsdetails (Vorschlag, im Rahmen der Entscheidung)

- In der Oberfläche heißen die Werte „Mitglied“ und „Nichtmitglied“. Im Code entsprechen sie dem Enum `AttendeeType` mit `member` und `non_member`.
- **Begriffsklärung:** „Gast“ bezeichnet eine mit angemeldete Begleitperson, **nicht** ein Nichtmitglied. Ein Gast kann Mitglied oder Nichtmitglied sein. Ob jemand Gast ist, ergibt sich aus der Buchung (er ist nicht die anmeldende Person), nicht aus der Teilnehmerart.
- Das Dropdown zeigt nur die Teilnehmerarten, die für das Event zugelassen sind. Ist nur eine zugelassen, entfällt das Dropdown und der Wert ist fest gesetzt.
- Bei Login wird für den ersten Teilnehmer „Mitglied“ vorbelegt, sofern das Konto als Mitglied erkennbar ist. Der Wert bleibt änderbar.
- Die Teilnehmerart ist ein festes Feld des Teilnehmers im Datenmodell, kein dynamisches Zusatzfeld (§8).

## Folgen

- Die Teilnehmerart ist eine Angabe der Person und nicht belegt.
- Sollen einzelne Events Plätze ausschließlich für Mitglieder freihalten, reicht die Selbstauskunft dafür nur eingeschränkt. Dann wäre eine Prüfung neu zu entscheiden.
- Da keine Mitgliedsnummer gespeichert wird, entfällt dieses personenbezogene Datum ganz (Datensparsamkeit).
