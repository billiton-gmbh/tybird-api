# Status

Stand 27.11.2026.

| Bereich | Stand |
|---|---|
| `POST /tickets`, `GET /tickets`, `GET /tickets/{id}` | in Betrieb |
| `POST /tickets/{id}/antworten`, `POST /tickets/{id}/bestaetigen` | in Betrieb |
| `anhaenge` als Data-URL in Anfrage und Antwort | in Betrieb |
| `POST /anhaenge` (signierter Upload) mit `anhang_ids` | in Betrieb |
| `POST /kunden` (Kundenabgleich) | in Betrieb |
| Mails an den Melder (Eingegangen, Antwort, Rückfrage, Gelöst) | in Betrieb seit 26.11.2026, minütlich |
| Antworten des Melders per Mail | in Betrieb: Antworten auf eine Helpdesk-Mail landen am Ticket |

## Was Produkte heute nutzen

Ein Produkt kann mit `anhaenge` als Data-URL beginnen (zum Beispiel Bildschirmfotos, je Datei bis 5 MB) und den signierten Upload erst später nutzen. Beides darf in derselben Anfrage zusammen vorkommen; die API unterscheidet die Wege nicht.

## Nicht Teil von v1.1

- Löschen von Anfragen oder Anhängen über die API.
- Rückruf (Webhook) bei neuen Antworten: Antworten werden mit `GET /tickets/{id}` abgeholt.
