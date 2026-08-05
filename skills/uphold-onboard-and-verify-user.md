---
name: Onboard and verify an Uphold user
description: Create an individual user on the Uphold Enterprise platform, drive the required KYC processes to `ok`, and confirm the user is allowed to transact before you attempt any money movement.
api: openapi/uphold-core-api-openapi.json
operations:
  - core.create-oauth2-token
  - core.list-countries
  - core.list-terms-of-service
  - core.accept-terms-of-service
  - core.create-user
  - core.get-kyc-overview
  - core.update-kyc-profile
  - core.update-kyc-identity
  - core.update-kyc-proof-of-address
  - core.update-kyc-customer-due-diligence
  - core.list-capabilities
generated: '2026-08-05'
method: generated
source: openapi/uphold-core-api-openapi.json + https://developer.uphold.com/developer-guides/user-onboarding/individual/via-api
---

# Onboard and verify an Uphold user

Every operationId below was verified against `openapi/uphold-core-api-openapi.json`. Do not invent
operations — if a step is not in the spec, it does not exist on this API.

## 1. Get a token

Call `core.create-oauth2-token` (`POST /core/oauth2/token`). Uphold supports **only** the OAuth 2.0
client-credentials grant — there is no API-key auth. Send the token as `Authorization: Bearer {token}`.

Your client is either **organization-wide** (defaults to the `client` subject; can act for any user in
the org if it holds the `core.users:act-on-behalf-of` scope, via the `X-On-Behalf-Of: user {userId}`
header) or **single-user** (bound to one user, no header needed). Scopes are per-client; a call missing
a scope fails `403` with code `token_insufficient_scopes`.

## 2. Check the country is supported, then take Terms of Service

- `core.list-countries` / `core.get-country` — confirm the residency country is supported and read its
  KYC requirements before you collect anything.
- `core.list-terms-of-service` then `core.accept-terms-of-service` — acceptance is per ToS `code`.
  Passing the wrong one fails `409 terms_of_service_mismatch`.

## 3. Create the user

`core.create-user` (`POST /core/users`). As of the 2026.07.14 release you may pass `fullName`,
`birthdate`, `address`, `primaryCitizenship` and `phone` inline — the profile fields complete the
`profile` KYC process up front, and `profile` now also covers address. Conflicts you must handle:
`email_already_exists`, `phone_already_exists`, `invalid_characters`, `date_invalid` (under 18 / over
100), `country_not_supported`, `subdivision_not_supported`, `subdivision_required`,
`po_boxes_not_allowed`, `postal_code_invalid`.

Always send the end-user context headers (`X-Uphold-User-Ip`, `X-Uphold-User-Agent`,
`X-Uphold-User-Country`, …). Some endpoints require them to function correctly.

## 4. Drive the KYC processes

Read state with `core.get-kyc-overview` (`GET /core/kyc`) — note this is the *current* user's overview,
resolved from the token subject / `X-On-Behalf-Of` header, not a `{userId}` path parameter.

Then PATCH only the processes the overview reports as required:

| Process | Operation |
|---|---|
| Profile (name, DOB, citizenship, address) | `core.update-kyc-profile` |
| Email | `core.update-kyc-email` |
| Phone | `core.update-kyc-phone` |
| Identity document | `core.update-kyc-identity` |
| Proof of address | `core.update-kyc-proof-of-address` |
| Customer due diligence | `core.update-kyc-customer-due-diligence` |
| Enhanced due diligence | `core.update-kyc-enhanced-due-diligence` |
| Crypto risk assessment | `core.update-kyc-crypto-risk-assessment` |
| Self-categorization statement | `core.update-kyc-self-categorization-statement` |
| Tax details | `core.update-kyc-tax-details` |

`core.update-kyc-address` is marked **deprecated** in the spec — migrate to `core.update-kyc-profile`.

Documents are uploaded first with `core.create-file` (check `core.list-files-settings` for allowed
content types and limits; violations fail `file_content_type_not_allowed`, `max_files_limit_exceeded`).

## 5. Wait on events, do not poll hard

Subscribe to the `core.kyc.*.status-changed` webhooks (13 of them, plus `core.kyc.screening.*` and
`core.kyc.risk.*`) — see `asyncapi/uphold-core-webhooks.yml`. Verify every delivery with the Svix
`Webhook-Id` / `Webhook-Timestamp` / `Webhook-Signature` headers before processing it.

KYC status can revert to `pending` on periodic review (an expiring ID). Treat re-verification as a
normal state, not an error.

## 6. Confirm capabilities before offering a product

`core.list-capabilities` / `core.get-capability`. KYC being `ok` is **not** the same as being allowed to
deposit, trade or withdraw. A blocked user fails every non-GET call with `409 operation_not_allowed` and
`reasons: [user-blocked]`; a capability constraint surfaces as `409 user_capability_failure`.

## 7. Test it in Sandbox first

Use the Sandbox host `https://api.enterprise.sandbox.uphold.com`. Test helpers
(`core.simulate-bank-deposit`, `core.simulate-crypto-deposit`, `core.skip-assets-cooldowns`) exist only
there and return `404` in Production. See `sandbox/uphold-sandbox.yml`.

## Conventions that apply throughout

- **No idempotency contract.** Uphold documents no idempotency key on any endpoint. Do not assume a
  retried `POST /core/users` or `POST /core/transactions` is safe — reconcile by reading before you retry.
- Errors: branch on the `code` field, not the HTTP status. Catalog: `errors/uphold-problem-types.yml`.
- Pagination: `perPage` plus a `pagination` object with `first`/`next`/`previous` URLs. Follow the URLs.
- Rate limits: `429` with a `Reset-After` header in seconds. Back off.
- Every response carries `X-Uphold-Request-Id`; log it — support asks for it.
