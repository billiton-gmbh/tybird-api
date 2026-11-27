# Helpdesk-Integration v1.1 - Paket für Produktteams

Stand 27.11.2026, Support-API 1.1, Dokumentation 2.6.0.

Dieses Paket enthält alles, was ein Produkt braucht, um Anfragen seiner Nutzer an den tybird-Helpdesk zu geben und Antworten zurückzuholen.

| Datei | Inhalt |
|---|---|
| `openapi.yaml` | Die Support-API als OpenAPI-Auszug: Tickets, Antworten, Bestätigen, Anhänge, Kundenabgleich |
| `STATUS.md` | Was heute in Betrieb ist und was nicht |
| `CHANGELOG.md` | Änderungen der Dokumentation 2.0.0 bis 2.6.0 |
| `GRENZEN.md` | Größen, Mengen und Drosseln mit den Antwortcodes |
| `MAILS.md` | Welche Mails der Melder bekommt und wann |
| `KUNDEN.md` | Kundenliste abgleichen, `extern_id`, Admin-Adressen |

## Einstieg

1. Den Support-Schlüssel (`tyb_live_…`) erhalten Sie von billiton, nicht aus diesem Paket. Er steht im Kopf `Authorization: Bearer …` (siehe `securitySchemes` in `openapi.yaml`) und gehört nur auf Ihren Server, nie in eine App oder einen Browser.
2. Kunden abgleichen (`POST /kunden`), damit Anfragen der richtigen Firma zugeordnet werden.
3. Anfragen anlegen (`POST /tickets`), Antworten lesen (`GET /tickets/{id}`) und schreiben (`POST /tickets/{id}/antworten`).
4. Dateien: kleine Bilder und Dokumente direkt als Data-URL in `anhaenge`, größere über `POST /anhaenge` (signierter Upload) und `anhang_ids`.
