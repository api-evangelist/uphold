---
name: Quote and execute an Uphold transaction
description: Move value on Uphold using the quote-then-commit model — deposits, withdrawals, trades and sends — including requirements handling, on-hold requests for information, and settlement monitoring.
api: openapi/uphold-core-api-openapi.json
operations:
  - core.create-oauth2-token
  - core.list-accounts
  - core.create-account
  - core.list-default-accounts
  - core.get-account-deposit-method
  - core.setup-account-deposit-method
  - core.create-external-account
  - core.list-external-accounts
  - core.create-quote
  - core.create-transaction
  - core.get-transaction
  - core.list-transactions
  - core.list-transaction-requests-for-information
  - core.update-transaction-request-for-information
generated: '2026-08-05'
method: generated
source: openapi/uphold-core-api-openapi.json + https://developer.uphold.com/rest-apis/core-api/transactions/introduction
---

# Quote and execute an Uphold transaction

All operationIds verified against `openapi/uphold-core-api-openapi.json`.

## The model

A transaction is defined by an **origin** node, a **destination** node and a direction. You never create
a transaction from raw amounts — you create a **quote** first (`core.create-quote`,
`POST /core/transactions/quote`), then commit it (`core.create-transaction`, `POST /core/transactions`).
This is an RFQ model: the quote carries the rate, the fees and an expiry.

## 1. Have somewhere to move value from and to

- `core.list-accounts` / `core.list-default-accounts` — an account holds the balance of exactly one
  asset. `core.create-account` if the asset the user needs is missing.
- **Inbound external money** (a bank/card/APM pull, or an on-chain deposit): configure the deposit
  method with `core.setup-account-deposit-method`, then read the routing details or generated network
  address with `core.get-account-deposit-method`.
- **Outbound to a bank, card or PayPal**: register the destination first with
  `core.create-external-account`, then confirm status with `core.list-external-accounts` /
  `core.get-external-account`. Duplicate registrations fail `409 duplicate`; a bad IBAN/card fails
  `invalid_number`; an unsupported network on the destination fails `transaction_node_invalid`.
- **Outbound to a crypto address**: validate it first with `core.validate-network-address` — an invalid
  address fails `crypto_address_invalid`, and on-chain sends are irreversible.

## 2. Quote

`core.create-quote`. Read the response before showing anything to the user:

- the rate and fees to display,
- the **expiry** — quotes go stale in minutes; re-quote rather than commit late,
- the **requirements** array. A non-empty requirements array means the quote cannot be committed yet:
  FATF Travel Rule information, 3-D Secure authorization on a card, or an APM authorization. Satisfy
  each requirement and carry the fulfillment into the commit.

Common `409`s here: `insufficient_balance`, `transaction_amount_invalid` (limits or unsettled funds),
`asset_not_supported`, `asset_pair_not_supported`, `asset_feature_not_available`,
`rail_feature_not_available`, `rail_constraint_violated`, `remitter_and_recipient_cannot_match`,
`user_capability_failure`.

## 3. Commit

`core.create-transaction` with the quote id plus any fulfillments. **There is no idempotency key on this
endpoint.** If the call times out, do not blind-retry a money movement — call `core.list-transactions`
or `core.get-transaction` and reconcile first.

## 4. Watch it settle

Subscribe to `core.transaction.created` and `core.transaction.status-changed`
(`asyncapi/uphold-core-webhooks.yml`), or poll `core.get-transaction`. Verify the Svix signature on
every delivery.

## 5. Handle on-hold and requests for information

A transaction can pause on-hold pending an RFI (commonly Travel Rule on a crypto deposit or withdrawal):

1. `core.list-transaction-requests-for-information`
2. `core.get-transaction-request-for-information`
3. `core.update-transaction-request-for-information` to submit the answer

Uphold also ships a Travel Rule Widget if you would rather not build the collection UI — see
`components/uphold-components.yml`.

## 6. Reconcile

`core.list-account-transactions` and `core.list-transactions` for the ledger;
`core.get-transactions-statement` and `core.get-portfolio-statement` for period statements;
`core.get-portfolio-overview` / `core.get-portfolio-performance` for position and P&L.

Attach your own reference with the metadata operations (`core.set-metadata`, `core.get-metadata`) —
metadata supports optimistic concurrency via `If-Match` (`412 precondition_failed`) and JSON Patch
(`422 json_patch_failed`).

## Testing

Sandbox host `https://api.enterprise.sandbox.uphold.com`. `core.simulate-bank-deposit` and
`core.simulate-crypto-deposit` fake the inbound leg; `core.skip-assets-cooldowns` removes asset
cooldowns. All three return `404` in Production.
