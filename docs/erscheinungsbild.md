# DFK Events – Erscheinungsbild (Vorlage: heutige Seiten)

Stand: 18.11.2026 · Grundlage: Screenshots `src/screenshots/` (Übersicht und Detailseite auf dfk.eu, The Events Calendar)

Ziel (A8): das bisherige Erscheinungsbild möglichst beibehalten. V1 mit eigenen Plugin-Templates; Shortcodes für Elementor folgen in Schritt 2 (zur Not wird HTML in einem Elementor-Container eingebunden).

## 1. Übersicht

Untereinander angeordnete Karten (eine Spalte, keine Kachel-Raster), rot umrandet, abgerundet:

| Bereich | Inhalt |
|---|---|
| Links | Beitragsbild, oben links Datums-Badge (Monat kurz, z. B. „NOV.“, darunter Tag groß) in DFK-Rot |
| Rechts oben | Veranstalter als rote Plakette (z. B. „RG Süd“), daneben Uhr-Symbol + „18. November @ 14:20 - 16:00“ |
| Rechts Mitte | Titel (groß), Kurzbeschreibung (Auszug, gekürzt mit „…“) |
| Rechts unten | Button „Jetzt anmelden →“ (rot umrandet) – neu: bei voller Veranstaltung „Auf die Warteliste“, nach Anmeldeschluss kein Button |

Ergänzungen gegenüber heute: Filterleiste (Zeitraum, Veranstalter, PLZ/Umkreis) über der Liste, Entfernung in km an der Karte, Kennzeichnung „Online“. Kompakte Listenansicht umschaltbar.

Mobil: Bild über dem Text, volle Breite.

## 2. Detailseite

| Bereich | Inhalt heute | Mit DFK Events |
|---|---|---|
| Kopf (grauer Balken) | Titel in Rot; drei Spalten Datum / Zeit / Lokation | gleich; Lokation ohne „Germany“ |
| Rechts oben | „Teilen Sie dieses Event“: LinkedIn, Facebook, WhatsApp, Drucken | gleich, als einfache Links ohne Tracking-Skripte |
| Rechte Spalte | Bild; Buttons „+ Google Calendar“, „+ ICal Export“ | gleich (Kalenderexport aus AGENTS.md §11) |
| Rechte Spalte | Box „Veranstalter“: Name, E-Mail | Name aus `dfk_unit`, E-Mail siehe O7 |
| Hauptteil | „Veranstaltungs Details“: Beschreibung | gleich |
| Hauptteil | Anmeldeschluss steht als Text in der Beschreibung | **strukturiert** aus `registration_closes_at`, plus Anzeige freie Plätze / Warteliste |
| Hauptteil | Hinweis zu Foto- und Filmaufnahmen steht in jeder Beschreibung | siehe O6 |
| Hauptteil | Adresse mit Kartensymbol, „+Google Map“, eingebettete Google-Karte | siehe O5 |
| Hauptteil | – | **Anmeldeformular** (neu im Plugin) |
| Fuß | Vor/Zurück-Links zur vorherigen/nächsten Veranstaltung | gleich |

Farben und Schriften kommen aus dem Astra-Child-Theme; das Plugin bringt nur eine kleine eigene CSS-Datei für Karten, Badge und Formular mit.

## 3. Gestaltungsfragen (entschieden 18.11.2026)

| # | Thema | Entscheidung |
|---|---|---|
| O5 | **Google-Karte** | Bleibt eingebettet wie heute. Auf dfk.eu ist **Borlabs Cookie** mit Content Blocker für Google, YouTube usw. vorgeschaltet. Das Plugin gibt eine normale Google-Maps-Einbettung (`iframe` mit den gespeicherten Koordinaten, ohne API-Schlüssel) aus, damit der Borlabs-Content-Blocker sie erkennt und erst nach Einwilligung lädt. Dazu der Link „+Google Map“ |
| O6 | **Foto-/Film-Hinweis** | Einmal in den Einstellungen hinterlegt, automatisch unter jeder Veranstaltung und im Anmeldeformular angezeigt; pro Veranstaltung abschaltbar |
| O7 | **Veranstalter-E-Mail** | Optionale E-Mail je Veranstalter (Term-Meta `email` an `dfk_unit`); fehlt sie, gilt die zentrale Adresse aus den Einstellungen |

Hinweis: Die Screenshots liegen in `src/`. Dort entsteht später der Plugin-Code; vorgeschlagen ist, sie nach `docs/design-referenz/` zu verschieben.
