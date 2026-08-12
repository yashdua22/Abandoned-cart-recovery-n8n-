# Abandoned-cart-recovery-n8n-
# Abandoned Cart Recovery (n8n)

An n8n workflow that recovers abandoned shopping carts. When a cart sits untouched for a set time, it checks whether the order was actually completed — and only if it wasn't, sends a personalised discount over WhatsApp and email.

Built without a live store. A Google Sheet stands in for the store database, and two webhooks simulate the `cart created` and `order paid` events a real Shopify or WooCommerce store would fire.

## What it demonstrates

- **The Wait node** — pausing an execution for hours and resuming it later
- **State checking** — re-reading the source of truth after the wait instead of trusting stale data
- **Conditional branching** — an IF node that decides whether the discount is warranted
- **Two independent triggers** sharing one datastore
- **LLM-generated copy** — Gemini writes the email body from structured cart data

## How it works

```
Webhook (cart created)
  └─> Sheets: append row, status = abandoned
       └─> Wait (2 hours)
            └─> Sheets: look up that cart_id  ← re-reads current state
                 └─> IF status is still "abandoned"
                      ├─ true  ─> WhatsApp ─> LLM ─> Gmail ─> Sheets: mark recovery sent
                      └─ false ─> stop (customer already paid)

Webhook (cart paid)                              ← separate trigger, same sheet
  └─> Sheets: update row, status = completed
       └─> Sheets: read row
            └─> LLM ─> Gmail: order confirmation
```

The two branches never connect on the canvas. They communicate only through the sheet — which is exactly what makes the state check meaningful.

## Google Sheet columns

| cart_id | name | email | phone | items | total | status | updated_at | recovery-mail | sent-at |
|---|---|---|---|---|---|---|---|---|---|

`status` is the state flag. It holds `abandoned` or `completed`.

## Setup

1. Import `abandoned-cart-recovery.json` into n8n.
2. Create a Google Sheet with the columns above and replace every `YOUR_GOOGLE_SHEET_ID` placeholder.
3. Connect your own credentials in each node:
   - Google Sheets (OAuth2)
   - Gmail (OAuth2)
   - Google Gemini API
4. For WhatsApp, replace the Twilio placeholders in the HTTP Request node:
   - `YOUR_TWILIO_ACCOUNT_SID`
   - `YOUR_BASE64_ENCODED_SID_AND_AUTH_TOKEN`
   - `YOUR_TWILIO_SANDBOX_NUMBER`, `YOUR_RECIPIENT_NUMBER`, `YOUR_TWILIO_TEMPLATE_SID`
5. Activate the workflow and copy both production webhook URLs.

The Wait node ships at 10 minutes for testing. Change it to 2 hours for production use.

## Testing

Send this to the `cart-created` webhook:

```json
{
  "cart_id": "CART001",
  "name": "Rahul",
  "email": "you@example.com",
  "phone": "91XXXXXXXXXX",
  "items": "Blue kurta x1",
  "total": 1299
}
```

Wait for the timer. You should receive the discount email and WhatsApp message, and the row should flip to `discount-message-sent`.

Then repeat with `CART002`, but POST to the `cart-paid` webhook before the timer expires. This time nothing should send — the IF node takes the false branch, and you get an order confirmation instead.

Passing both runs is the proof the state check works.

## Notes and limitations

- Twilio's WhatsApp sandbox requires the recipient to opt in first. Production WhatsApp needs a Meta-approved message template.
- Every recovery email costs one Gemini call. For real deployments a static template is cheaper and the copy barely varies.
- No retry or dead-letter handling on API failures.
- No credentials are stored in this repo. All secrets are placeholders.

## Tech

n8n · Google Sheets · Gmail · Google Gemini · Twilio WhatsApp API
