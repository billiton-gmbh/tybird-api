# Grenzen

Alle Grenzen gelten je Support-Schlüssel. Überschreitungen antworten mit dem genannten Code und einer Meldung im Feld `fehler`.

## Mengen je Stunde

| Was | Grenze | Zähler | Code |
|---|---|---|---|
| Neue Anfragen (`POST /tickets`) | 20 | je Melder | 429 |
| Antworten und Bestätigen | 60 | je Melder | 429 |
| Lesen (`GET`) | 600 | je Melder | 429 |
| Signierter Upload (`POST /anhaenge`) | 60 | je Melder | 429 |
| Kundenabgleich (`POST /kunden`) | 20 | je Produkt | 429 |

## Dateien

| Weg | Grenze | Code |
|---|---|---|
| `bild` (PNG oder WebP, Data-URL) | 2 MB | 413 |
| `anhaenge`, je Data-URL | 5 MB | 413 |
| `anhaenge`, alle Data-URLs einer Anfrage zusammen | 20 MB; darüber `POST /anhaenge` nehmen | 413 |
| Signierter Upload: Bild | 5 MB | 413 |
| Signierter Upload: Video | 100 MB | 413 |
| Signierter Upload: andere Dateien | 20 MB | 413 |
| Anhänge je Anfrage | 100 | 400 |

Die Upload-URL aus `POST /anhaenge` ist für einen einzigen `PUT` gedacht und läuft nach zwei Stunden ab. Danach eine neue anfordern.

## Kundenabgleich

1 bis 500 Einträge je Aufruf, `extern_id` bis 120 Zeichen, `name` bis 160 Zeichen, bis zu 20 Admin-Adressen je Kunde. Siehe `KUNDEN.md`.

## Weitere Codes

| Code | Bedeutung |
|---|---|
| 400 | Pflichtfeld fehlt, ist zu lang oder hat die falsche Form; Dateien in unbekannten Feldern |
| 401 | Support-Schlüssel fehlt oder ist ungültig |
| 404 | Anfrage nicht gefunden oder gehört nicht zu diesem Produkt |
| 503 | Wartung; später erneut senden |
