# AfriCard — Monerails integration

## Supabase Edge Function

Function: `africard-monerails`

Webhook URL:

`https://hytgdeihbaqmgynojmrk.supabase.co/functions/v1/africard-monerails?action=webhook`

## Required Supabase Edge Function secrets

Set these in Supabase → Project Settings → Edge Functions → Secrets:

- `MONERAILS_API_KEY`: API key from the Monerails business account. Use an Owner or Finance key because cardholder verification and card issuance require those roles. Use a test key for test mode, then replace it with the live key after KYB approval.
- `MONERAILS_WEBHOOK_SECRET`: signing secret (`whsec_...`) returned when the webhook endpoint is created in Monerails. Never put it in the browser or GitHub.

Do not send either secret through chat or commit it to the repository.

## Monerails webhook endpoint

Create an HTTPS webhook endpoint in Monerails Developers → Webhooks with the URL above. Subscribe to:

- `collection.*`
- `card.*`
- `cardholder.*`

Monerails sends `Monerail-Signature`; the Edge Function verifies the HMAC-SHA256 signature against the raw request body and rejects stale or invalid signatures. Events are deduplicated by event ID.

## Supported customer flows

- Wallet deposit: `create_collection`, `check_collection`, `authorize_collection`, `resend_collection_otp`.
- Issuer cardholder onboarding: `create_cardholder`. Didit must first approve the AfriCard profile. Monerails separately requires the cardholder's identity details and document image(s); the form sends the images directly to Monerails and does not store the raw images in AfriCard.
- Card issue: `create_card`. Server-side wallet debit is atomic and idempotent. A confirmed issuer failure refunds the debit. An uncertain provider response stays pending until webhook reconciliation.
- Card funding: `fund_card`. The amount is converted from XOF to USD cents, debited atomically, then sent to Monerails. Confirmed provider failures are refunded.
- Card details: `card_details`. Full number/CVC are fetched on demand, never stored, and displayed temporarily only.
- Card transaction history and 5% AfriCard fee accruals are recorded from `card.transaction` webhooks. If the wallet has insufficient funds, the fee remains pending and is collected from a later successful wallet deposit.

## Pricing and settlement rules

- Customer card order: USD 6 total.
- Issuer call: USD 2 total — USD 1 issuer fee plus USD 1 initial card balance.
- AfriCard creation fee: USD 5; expected gross margin before collection costs, FX spread, taxes and other costs: USD 4.
- No AfriCard fee on deposits or card funding.
- 5% AfriCard fee only on successful card purchases.
- Wallet deposits are credited net of the Monerails collection fee. The provider fee is not an AfriCard fee and must be shown to the customer.

## Production prerequisites

- Monerails business KYB approved and live API key enabled.
- Monerails cards balance funded. Its card documentation currently specifies a minimum crypto top-up of USD 100.
- Monerails webhook created and its signing secret stored in Supabase.
- Didit Live account funded and its webhook secret stored in Supabase.
- Complete test-mode end-to-end checks before switching to live keys.