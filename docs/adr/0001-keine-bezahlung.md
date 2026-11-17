# ADR 0001 – Keine Bezahlung, reines RSVP-System

Datum: 17.11.2026 · Status: **Angenommen** (Captain)

## Kontext

DFK Events ersetzt The Events Calendar Pro und Event Tickets Plus. Event Tickets Plus kann kostenpflichtige Tickets verkaufen. Zu klären war, ob DFK Events das ebenfalls können muss.

## Entscheidung

Alle Veranstaltungen sind kostenfrei. DFK Events ist ein reines Anmeldesystem (RSVP). Zahlungen, Preise, Rechnungen und Zahlungsanbieter sind nicht Bestandteil des Plugins.

## Folgen

- Kein Zahlungsanbieter, keine Bestellungen, keine Rechnungen, kein WooCommerce.
- Eine Buchung ist sofort gültig. Es gibt keinen Zustand „wartet auf Zahlung“.
- Die Begriffe im Code lauten „Buchung“ bzw. „Anmeldung“ und „Teilnehmer“, nicht „Ticket“ oder „Bestellung“. Der spätere QR-Code für den Check-in ist dem Teilnehmer zugeordnet.
- Kommt später doch Bezahlung hinzu, wird das als eigenes Modul an den `BookingService` angebunden. Das Datenmodell sieht dafür bewusst nichts vor.
