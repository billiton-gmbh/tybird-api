# Status

Stand 27.11.2026.

| Bereich | Stand |
|---|---|
| `POST /tickets`, `GET /tickets`, `GET /tickets/{id}` | in Betrieb |
| `POST /tickets/{id}/antworten`, `POST /tickets/{id}/bestaetigen` | in Betrieb |
| `anhaenge` als Data-URL in Anfrage und Antwort | in Betrieb |
| `POST /anhaenge` (signierter Upload) mit `anhang_ids` | in Betrieb |
| `POST /kunden` (Kundenabgleich) | in Betrieb |
| Mails an den Melder: Antwort, Rückfrage, Gelöst | in Betrieb seit 27.11.2026, minütlich |
| Mail an den Melder: Eingangsbestätigung („Eingegangen") für jede neue Anfrage | vorbereitet, wird nach Freigabe eingeschaltet |
| Antworten des Melders per Mail | in Betrieb: Antworten auf eine Helpdesk-Mail landen am Ticket |

## Status einer Anfrage

Die API nennt sechs Status. Zeigen Sie dem Melder genau diesen Status.

| Status | Bedeutung |
|---|---|
| `eingegangen` | Neu, noch nicht bearbeitet |
| `in_arbeit` | Das Team klärt, beschreibt oder entwickelt |
| `rueckfrage` | Das Team wartet auf eine Antwort des Melders |
| `geplant` | Zurückgestellt, wird später wieder aufgenommen |
| `erledigt` | Gelöst, die Antwort liegt vor |
| `geschlossen` | Vom Melder bestätigt oder vom Team abgeschlossen |

## Übergänge aus Sicht des Melders

| Der Melder ... | Aufruf | bei Status | danach |
|---|---|---|---|
| antwortet | `POST /tickets/{id}/antworten` | `rueckfrage` oder `erledigt` | `in_arbeit` |
| antwortet | `POST /tickets/{id}/antworten` | alle anderen | unverändert; die Antwort steht am Ticket |
| klickt „Hat geholfen" | `POST /tickets/{id}/bestaetigen` mit `hilfreich: true` | `erledigt` | `geschlossen` |
| fragt nach („Hat nicht geholfen") | `POST /tickets/{id}/bestaetigen` mit `hilfreich: false` | `erledigt` | `in_arbeit` |

In anderen Status speichert `bestaetigen` nur die Rückmeldung; der Status bleibt. „Geschlossen" wird eine Anfrage nie von selbst.

## Was Produkte heute nutzen

Ein Produkt kann mit `anhaenge` als Data-URL beginnen (zum Beispiel Bildschirmfotos, je Datei bis 5 MB) und den signierten Upload erst später nutzen. Beides darf in derselben Anfrage zusammen vorkommen; die API unterscheidet die Wege nicht.

## Nicht Teil von v1.1

- Löschen von Anfragen oder Anhängen über die API.
- Rückruf (Webhook) bei neuen Antworten: Antworten werden mit `GET /tickets/{id}` abgeholt.
