---
name: Reuse a third-party KYC verification on Uphold
description: Bring an existing Sumsub or Veriff verification into Uphold with the KYC Connector API so an already-verified user does not have to be re-verified, and watch the ingestion resolve into Uphold KYC processes.
api: openapi/uphold-kyc-connector-api-openapi.json
operations:
  - kyc-connector.set-veriff-config
  - kyc-connector.get-veriff-config
  - kyc-connector.create-sumsub-ingestion
  - kyc-connector.list-sumsub-ingestions
  - kyc-connector.get-sumsub-ingestion
  - kyc-connector.create-veriff-ingestion
  - kyc-connector.list-veriff-ingestions
  - kyc-connector.get-veriff-ingestion
  - core.get-kyc-overview
  - core.list-capabilities
generated: '2026-08-05'
method: generated
source: openapi/uphold-kyc-connector-api-openapi.json + https://developer.uphold.com/rest-apis/kyc-connector-api/introduction
---

# Reuse a third-party KYC verification on Uphold

All operationIds verified against `openapi/uphold-kyc-connector-api-openapi.json` and
`openapi/uphold-core-api-openapi.json`.

## When to reach for this

You already verify users in **Sumsub** (Reusable KYC) or **Veriff**, and you do not want to make them
repeat identity capture to transact on Uphold. The KYC Connector ingests the provider's payload and maps
it onto Uphold KYC processes. If you do not already run one of those two providers, onboard through the
Core API instead — see `skills/uphold-onboard-and-verify-user.md`.

## 1. Configure the provider (Veriff only)

`kyc-connector.set-veriff-config` (`PUT /kyc-connector/veriff/config`) stores your organization's Veriff
Station configuration; read it back with `kyc-connector.get-veriff-config`. Bad parameters fail
`400 request_invalid`. Sumsub has no config endpoint — configuration is handled during onboarding with
your account manager.

Scopes: `kyc-connector.veriff.config:write` / `kyc-connector.veriff.config:read`.

## 2. Create the ingestion

- Sumsub: `kyc-connector.create-sumsub-ingestion` (`POST /kyc-connector/sumsub/ingestions`), scope
  `kyc-connector.sumsub.ingestions:create`
- Veriff: `kyc-connector.create-veriff-ingestion` (`POST /kyc-connector/veriff/ingestions`), scope
  `kyc-connector.veriff.ingestions:create`

The call is made in the context of the user (organization-wide clients send
`X-On-Behalf-Of: user {userId}`), so the user must exist on Uphold first.

## 3. Track it

Ingestion is asynchronous. Subscribe to the four KYC Connector webhooks — `*.ingestion.created` and
`*.ingestion.status-changed` for both providers (`asyncapi/uphold-core-webhooks.yml`) — or poll
`kyc-connector.get-sumsub-ingestion` / `kyc-connector.get-veriff-ingestion`.
`kyc-connector.list-*-ingestions` gives you the history for a user.

## 4. Confirm on the Uphold side

An ingestion succeeding is not the finish line. Read `core.get-kyc-overview` to confirm the mapped
processes actually moved to `ok`, and `core.list-capabilities` to confirm the user is cleared for the
products you want to offer. Processes the provider payload does not cover (customer due diligence,
crypto risk assessment, tax details) still have to be completed through the Core API.

## Gotchas

- Ingestion maps *provider* fields onto Uphold processes; a field the provider never collected cannot be
  mapped, and the corresponding Uphold process stays outstanding.
- There is no idempotency key. A retried create can produce a second ingestion — list first.
- Test in Sandbox (`https://api.enterprise.sandbox.uphold.com`) before Production.
