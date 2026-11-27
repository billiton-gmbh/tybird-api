# Mails an den Melder

Der Helpdesk schreibt dem Melder aus dem Postfach des Produkts. Die Adresse des Melders kommt aus der Anfrage (`melder.email`). Fehlt sie, gibt es keine Mail; die Anfrage bleibt trotzdem gültig.

| Anlass | Betreff | Inhalt |
|---|---|---|
| Anfrage angelegt (nach Freigabe, siehe STATUS.md) | `[T-1234] Eingegangen: <Titel>` | Bestätigung, dass die Anfrage aufgenommen ist |
| Antwort des Teams | `[T-1234] Antwort: <Titel>` | Der Antworttext |
| Rückfrage des Teams | `[T-1234] Rückfrage: <Titel>` | Der Text der Rückfrage; die Anfrage wartet auf den Melder |
| Anfrage gelöst | `[T-1234] Gelöst: <Titel>` | Hinweis auf die Erledigung mit dem Lösungstext |

- **Anrede:** je Produkt „Sie" oder „du", Vorgabe „Sie".
- **Link:** Ist `ruecksprung_url` gesetzt (nur `https://`), führt ein Link „Zur Anfrage" dorthin; `{id}` wird durch die Ticket-id ersetzt.
- **Antworten per Mail:** Die Antwortadresse trägt die Ticketnummer. Antwortet der Melder, landet die Mail am Ticket wie eine Antwort über die API. Mails von den Admin-Adressen des Kunden (`KUNDEN.md`) werden ebenso angenommen.
- **Keine Schleifen:** Automatische Mails tragen `Auto-Submitted: auto-generated`, Antworten des Teams `Auto-Submitted: no`.
- **Takt:** Mails gehen minütlich hinaus; zwischen Ereignis und Mail liegen in der Regel unter zwei Minuten.
- **Weitere Empfänger:** Das Team kann einer Anfrage weitere Kontakte zuordnen, die alle Mails oder nur die Lösungsmail in Kopie (CC) erhalten. Das steuert der Helpdesk, nicht die API.
