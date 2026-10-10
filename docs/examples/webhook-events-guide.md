# Guida ai webhook OpenWA: eventi, payload e risposte

Riepilogo pratico degli eventi webhook emessi da OpenWA, del formato delle consegne e di
come deve rispondere il ricevitore. Il riferimento completo è la sezione
[6.6 Webhook Events & Delivery Semantics](../06-api-specification.md#66-webhook-events--delivery-semantics)
di `06-api-specification.md`.

## 1. Registrazione

- Creazione: `POST /api/sessions/:sessionId/webhooks`
- Modifica: `PUT /api/sessions/:sessionId/webhooks/:id`
- Test: `POST /api/sessions/:sessionId/webhooks/:id/test`

```json
{
  "url": "https://tuo-server.com/webhook",
  "events": ["message.received", "message.ack", "poll.vote"],
  "secret": "chiave-di-firma-min-16-caratteri",
  "headers": { "X-Custom-Header": "valore" },
  "filters": {
    "conditions": [{ "field": "chatId", "operator": "is", "value": ["120363000000000000@g.us"] }]
  },
  "retryCount": 3
}
```

- `events`: se omesso vale `["message.received"]`. `"*"` sottoscrive tutto. Ricevi solo ciò
  che è elencato qui.
- `filters`: condizioni in AND. Attenzione: se il campo filtrato non esiste nel payload di un
  evento, un filtro `is`/`contains`/`equals` scarta l'evento (es. `sender is …` blocca tutti i
  `message.ack`, che non hanno mittente). Limita le sottoscrizioni con `events`, non con i filtri.
- `retryCount`: tentativi totali (0–5, default 3).

## 2. Formato della consegna

Ogni evento è una `POST` con corpo JSON:

```json
{
  "event": "message.received",
  "timestamp": "2026-02-02T10:00:00.000Z",
  "sessionId": "my-session",
  "idempotencyKey": "msg_my-session_3EB0ABC123_<webhookId>",
  "deliveryId": "dlv_550e8400-e29b-41d4-a716-446655440000",
  "data": {}
}
```

`timestamp` è l'istante di invio; l'orario dell'evento vero e proprio è dentro `data`.

| Header                     | Significato                                                                      |
| -------------------------- | -------------------------------------------------------------------------------- |
| `X-OpenWA-Event`           | Nome dell'evento                                                                 |
| `X-OpenWA-Idempotency-Key` | Stabile tra i retry della stessa occorrenza: **usala per deduplicare**           |
| `X-OpenWA-Delivery-Id`     | Uguale nei retry automatici, nuovo nei replay; non usarlo per deduplicare        |
| `X-OpenWA-Retry-Count`     | Numero del tentativo (`0` = primo)                                               |
| `X-OpenWA-Signature`       | `sha256=<hex>`: HMAC-SHA256 del corpo grezzo con il `secret` (solo se impostato) |

## 3. Come deve rispondere il ricevitore

- **`2xx` entro `WEBHOOK_TIMEOUT` (default 10 s)**: consegna riuscita.
- **Non-2xx, timeout o errore di rete**: retry con backoff esponenziale a partire da
  `WEBHOOK_RETRY_DELAY` (default 5 s), fino a esaurire `retryCount`. La idempotency key resta
  la stessa.
- **Tentativi esauriti**: l'evento finisce in `GET /api/webhooks/delivery-failures` e si può
  rinviare con `POST /api/webhooks/delivery-failures/redrive`. Il webhook resta attivo.
- **At-least-once**: lo stesso evento può arrivare più volte, quindi l'handler deve essere
  idempotente (deduplica su `X-OpenWA-Idempotency-Key`).

Consiglio: rispondi `200` subito ed elabora in modo asincrono. Verifica la firma sul corpo
**grezzo**, con confronto a tempo costante:

```javascript
const crypto = require('crypto');

function verify(rawBody, header, secret) {
  if (typeof header !== 'string') return false;
  const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(rawBody).digest('hex');
  const a = Buffer.from(header);
  const b = Buffer.from(expected);
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

Esempi completi Express/FastAPI: [webhook-signature-verification.md](webhook-signature-verification.md).

## 4. Catalogo eventi (`data`)

### Messaggi

**`message.received` / `message.sent`**: messaggio in arrivo o inviato da questa sessione.

```json
{
  "id": "3EB0ABC123",
  "from": "393330000000@c.us",
  "to": "393470000000@c.us",
  "chatId": "393330000000@c.us",
  "body": "ciao",
  "type": "chat",
  "timestamp": 1760000000,
  "isGroup": false,
  "kind": "individual",
  "fromMe": false,
  "media": { "mimetype": "image/jpeg", "data": "<base64>" },
  "contact": {}
}
```

- `timestamp` in secondi; `kind` ∈ `individual|group|channel|status|broadcast|unknown`.
- I media grandi (oltre ~768 KiB con i limiti di default) arrivano come
  `{ "mimetype": "…", "omitted": true, "sizeBytes": 123 }` e si scaricano da
  `GET /api/sessions/:sessionId/messages/:chatId/:messageId/media`.
- `senderPhone` compare solo per mittenti `@lid` e solo con `RESOLVE_LID_TO_PHONE=true`.

**`message.ack`**: aggiornamento dello stato di un messaggio inviato.

```json
{ "id": "3EB0ABC123", "messageId": "3EB0ABC123", "status": "read", "ack": 3, "chatId": "393330000000@c.us" }
```

`status` ∈ `pending|sent|delivered|read|failed`; `ack` è il valore intero legacy (deprecato).

**`message.failed`**: come `message.ack`, con `status: "failed"` e `ack: -1`. Viene inviato in
aggiunta all'ack.

**`message.revoked`**: messaggio cancellato.

```json
{
  "id": "…",
  "revokedId": "…",
  "chatId": "…",
  "from": "…",
  "to": "…",
  "type": "revoked",
  "body": "",
  "timestamp": 1760000000
}
```

Riconcilia su `revokedId`; usa `id` solo se `revokedId` manca.

**`message.reaction`**: reazione aggiunta, cambiata o rimossa.

```json
{ "messageId": "…", "chatId": "…", "reaction": "👍", "senderId": "…", "reactions": { "<senderId>": "👍" } }
```

`reaction` è vuota quando la reazione viene rimossa. `reactions` (snapshot completo) può
mancare: in quel caso è "sconosciuto", non "vuoto".

**`message.edited`**: testo o didascalia modificati.

```json
{
  "messageId": "…",
  "chatId": "…",
  "body": "testo nuovo",
  "senderId": "…",
  "from": "…",
  "to": "…",
  "fromMe": false,
  "isGroup": false,
  "type": "chat",
  "hasMedia": false,
  "timestamp": 1760000100
}
```

`timestamp` è l'istante della modifica, non della creazione.

### Sondaggi

**`poll.vote`**: qualcuno vota, cambia voto o lo ritira.

```json
{
  "pollId": "<id restituito da send-poll>",
  "chatId": "120363000000000000@g.us",
  "voterId": "393330000000@c.us",
  "selectedOptions": ["Sì"],
  "timestamp": 1791633600.123
}
```

- `selectedOptions` è la selezione **completa e attuale** del votante, non un incremento;
  `[]` significa voto ritirato.
- Per il conteggio: salva la selezione per `(sessionId, pollId, voterId)` e sostituiscila a ogni
  evento, ignorando quelli con `timestamp` più vecchio.
- Filtri applicabili: `sender` (corrisponde a `voterId`) e `chatId`.
- **Baileys**: il voto è cifrato e si decifra solo se il sondaggio originale è nel message store
  della sessione (vale per i sondaggi inviati tramite API). Altrimenti l'evento non parte e nei
  log compare un warning. I voti non generano `message.received`/`message.sent`.
- **whatsapp-web.js**: deriva dall'evento `vote_update` della libreria.

### Sessione

| Evento                   | Quando                                                        | `data`                                                                                                                                    |
| ------------------------ | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `session.status`         | Cambio di stato della sessione                                | `{ sessionId, status }` con `created`, `initializing`, `qr_ready`, `authenticating`, `ready`, `disconnected`, `action_required`, `failed` |
| `session.qr`             | Nuovo QR di pairing                                           | `{ sessionId, qr }`: `qr` è un PNG data URL, utilizzabile in `<img src>`                                                                  |
| `session.authenticated`  | Sessione collegata e pronta                                   | `{ sessionId, phone, pushName }`                                                                                                          |
| `session.disconnected`   | Disconnessione lato WhatsApp/engine (non per stop/logout API) | `{ sessionId, reason }`                                                                                                                   |
| `session.reconnect_loop` | Ogni 5 tentativi di riconnessione                             | `{ sessionId, attempts, nextDelayMs }`                                                                                                    |
| `session.restriction`    | Restrizione sull'account applicata o rimossa                  | `{ sessionId, active, kind, code, expiresAt }`                                                                                            |

### Presenza

**`presence.update`**

```json
{
  "sessionId": "…",
  "chatId": "…",
  "participants": [{ "id": "…", "state": "composing", "lastSeen": 1760000000 }],
  "groupOnlineCount": 3
}
```

`state` ∈ `available|unavailable|composing|recording|paused`. Solo Baileys e solo dopo
`POST /api/sessions/:sessionId/presence/subscribe`; vengono inviati solo i cambi effettivi.

### Gruppi

Forma comune: `{ groupId, actorId?, participantIds, timestamp }`.

| Evento               | Quando                                                                                                          |
| -------------------- | --------------------------------------------------------------------------------------------------------------- |
| `group.join`         | Partecipanti aggiunti o entrati                                                                                 |
| `group.leave`        | Partecipanti usciti o rimossi                                                                                   |
| `group.update`       | Oggetto, descrizione o impostazioni cambiati; include `changes: { subject?, description?, announce?, locked? }` |
| `group.join_request` | Richiesta di ingresso in un gruppo amministrato; `participantIds` sono i richiedenti                            |

Su Baileys gli eventi avvenuti offline arrivano con l'orario di ricezione.

### Chiamate

**`call.received`**: chiamata in arrivo (su whatsapp-web.js non è affidabile).

```json
{ "callId": "…", "from": "…", "isVideo": false, "isGroup": false, "timestamp": 1760000000 }
```

**`call.accepted` / `call.rejected` / `call.missed`**: esito della chiamata (solo Baileys).

```json
{
  "sessionId": "…",
  "callId": "…",
  "from": "…",
  "outcome": "missed",
  "isVideo": false,
  "isGroup": false,
  "timestamp": 1760000000
}
```

Il `callId` corrisponde a quello della `call.received` precedente.

### Stati / storie

**`status.received`**: un contatto pubblica uno stato.

```json
{
  "sessionId": "…",
  "statusId": "…",
  "contact": { "id": "…", "name": "…", "pushName": "…" },
  "type": "image",
  "caption": "…",
  "hasMedia": true,
  "mediaOmitted": true,
  "postedAt": 1760000000000,
  "expiresAt": 1760086400000
}
```

- Opt-in: arriva solo se `events` contiene `"status.received"` o `"*"`.
- `postedAt`/`expiresAt` sono in **millisecondi**.
- Il media non è mai incluso: si scarica da `GET /api/sessions/:sessionId/status/:statusId/media`.

## 5. Idempotency key

Le chiavi sono derivate dal contenuto e terminano sempre con `_<webhookId>`.

| Evento                                                                          | Chiave                                                                 |
| ------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `message.received` / `message.sent`                                             | `msg_{session}_{messageId}`                                            |
| `message.ack`                                                                   | `ack_{session}_{messageId}_{status}`                                   |
| `message.failed`                                                                | `failed_{session}_{messageId}_{status}`                                |
| `poll.vote`                                                                     | `vote_{session}_{pollId}_{voterId}_{timestamp}_{hash}`                 |
| `call.received`                                                                 | `call_{session}_{callId}`                                              |
| `call.accepted` / `call.rejected` / `call.missed`                               | `call_{session}_{callId}_{outcome}`                                    |
| `message.edited`, `message.reaction`, `session.*`, `group.*`, `presence.update` | includono un `occurredAt`, stabile tra i retry della stessa occorrenza |
| altri (es. `status.received`)                                                   | `evt_{evento}_{hash(data)}`                                            |
