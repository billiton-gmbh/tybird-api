# Kunden abgleichen

`POST /kunden` legt die Kunden Ihres Produkts an oder gleicht sie ab. Ein Kunde ist die Organisation, für die ein Melder schreibt (zum Beispiel ein Hotel oder eine Firma).

## Feld `extern_id`

- Ihre dauerhafte Kennung des Kunden, 1 bis 120 Zeichen. Sie ändert sich nie, auch nicht bei einer Umbenennung.
- Dieselbe Kennung steht in `POST /tickets` unter `kunde.extern_id`. Darüber wird jede Anfrage dem Kunden zugeordnet.
- Ein Abgleich mit bekannter `extern_id` ändert Name und Admin-Adressen; eine neue `extern_id` legt den Kunden an.

## Aufruf

```json
POST /kunden
{ "kunden": [
  { "extern_id": "hotel-4711", "name": "Hotel Beispiel", "admin_emails": ["leitung@beispiel.de"] }
] }
```

Eine reine Liste ohne Hülle (`[ {...}, {...} ]`) ist ebenso erlaubt. Je Aufruf 1 bis 500 Einträge, höchstens 20 Abgleiche je Stunde.

## `admin_emails`

Bis zu 20 Adressen je Kunde. Mails von diesen Adressen auf eine Helpdesk-Mail werden als Antwort am Ticket angenommen, auch wenn nicht der Melder selbst schreibt. Die Adressen werden nirgends angezeigt und nicht angeschrieben.

## Empfehlung

Einmal die vollständige Liste abgleichen, danach bei Änderungen nur die geänderten Kunden senden. Ein Kunde, den Sie nicht mehr senden, bleibt bestehen.
