---
name: Enterprise
description: Use when building financial applications that require user onboarding, KYC verification, money movement (deposits, withdrawals, trades), account management, or transaction processing. Reach for this skill when integrating bank transfers, card payments, crypto transfers, or when you need to manage user capabilities and compliance workflows.
metadata:
    mintlify-proj: enterprise
    version: "1.0"
---

# Uphold Enterprise API Suite

## Product summary

Uphold Enterprise API Suite is a modular REST API platform for building enterprise-grade financial applications. It provides core building blocks for user onboarding (KYC/KYB), account management, transaction processing (deposits, withdrawals, trades, transfers), and asset management across fiat and crypto. The platform uses OAuth2 authentication, webhook-based event delivery, and a quote-based transaction model. Key resources: Core API for users, accounts, transactions, and assets; KYC Connector API for third-party KYC integration; Widgets API for embedded UI components; Market Pulse API for asset data. Access the Postman workspace for pre-configured requests and test environments (Sandbox and Production). OpenAPI specs available for all APIs at https://uphold.github.io/enterprise-docs/.

## When to use

Use this skill when:
- Building user onboarding flows with KYC/KYB verification (individual or business users)
- Implementing money movement: deposits (bank, card, crypto, APM), withdrawals, trades, or transfers
- Managing user accounts, external accounts (bank/card links), and transaction history
- Integrating payment widgets for low-code deposit/withdrawal flows
- Handling compliance workflows: KYC status monitoring, capabilities verification, Terms of Service acceptance
- Processing transactions with quote-based flows (RFQ model) or external transactions
- Monitoring transaction status and handling on-hold transactions with Requests for Information (RFIs)
- Integrating with third-party KYC providers (Sumsub, Veriff) via KYC Connector
- Retrieving asset information, rates, networks, and rails for supported cryptocurrencies and fiat

## Quick reference

### Authentication
- **Grant type**: OAuth2 client credentials
- **Endpoint**: `POST /core/authentication/oauth2/token`
- **Header**: `Authorization: Bearer {accessToken}`
- **Subjects**: `client` (organization-wide), `user:individual`, `user:business`
- **Act on behalf**: Add `X-On-Behalf-Of: user {userId}` header for organization-wide clients

### Core API endpoints (by resource)
| Resource | Key endpoints |
|----------|---------------|
| **Users** | POST /core/users, GET /core/users/{id}, DELETE /core/users/{id} |
| **KYC** | GET /core/kyc/{id}/overview, PATCH /core/kyc/{id}/email, PATCH /core/kyc/{id}/identity, etc. |
| **Accounts** | POST /core/accounts, GET /core/accounts/{id}, PATCH /core/accounts/{id}, POST /core/accounts/{id}/deposit-method |
| **External Accounts** | POST /core/external-accounts, GET /core/external-accounts/{id}, DELETE /core/external-accounts/{id} |
| **Transactions** | POST /core/transactions/quotes, POST /core/transactions, GET /core/transactions/{id} |
| **Assets** | GET /core/assets, GET /core/assets/{id}, GET /core/assets/{id}/rates, GET /core/networks |
| **Capabilities** | GET /core/users/{id}/capabilities |
| **Terms of Service** | GET /core/terms-of-service, POST /core/users/{id}/terms-of-service/acceptance |

### Webhook events (subscribe via Enterprise Portal or REST API)
| Event | Trigger |
|-------|---------|
| `core.user.created` | User registered |
| `core.kyc.*.status-changed` | KYC process status changed (email, identity, address, etc.) |
| `core.account.created` | Account created |
| `core.account.balance-changed` | Account balance updated |
| `core.transaction.created` | Transaction initiated |
| `core.transaction.status-changed` | Transaction status changed (processing → completed/failed/on-hold) |
| `core.external-account.created` | External account linked |
| `core.external-account.status-changed` | External account status changed |

### Transaction node types
| Node type | Use case |
|-----------|----------|
| **account** | User's Uphold account (origin or destination) |
| **external-account** | Linked bank/card (origin or destination) |
| **crypto-address** | Blockchain address (origin or destination) |
| **bank-address** | Originating bank for push deposits (origin only) |
| **apm** | Alternative payment method like PayPal (origin or destination) |

### Transaction types (by origin/destination)
| Type | Origin | Destination |
|------|--------|-------------|
| **Deposit** | External (card, bank, crypto, APM) | Account |
| **Withdrawal** | Account | External (card, bank, crypto, APM) |
| **Trade** | Account (asset A) | Account (asset B) |
| **Transfer** | Account (asset X) | Account (asset X) |

### Payment Widget flows
| Flow | Purpose |
|------|---------|
| `select-for-deposit` | User selects payment method to fund account (card, bank, crypto) |
| `select-for-withdrawal` | User selects external account to withdraw to |
| `authorize` | 3DS card authorization or PayPal authorization |

### Common HTTP status codes
| Code | Meaning | Action |
|------|---------|--------|
| 400 | Bad Request | Check request schema, validate required fields |
| 401 | Unauthorized | Verify OAuth2 token, check credentials |
| 403 | Forbidden | Verify token scopes, check user permissions |
| 404 | Not Found | Verify resource ID exists |
| 409 | Conflict | Business logic error (user blocked, capability missing, etc.) |
| 429 | Rate Limited | Wait for `Reset-After` header duration, implement backoff |

## Decision guidance

### When to use REST API vs Payment Widget

| Scenario | Use REST API | Use Payment Widget |
|----------|-------------|-------------------|
| Full control over UI/UX | ✓ | |
| Quick integration, minimal frontend | | ✓ |
| Custom payment method filtering | | ✓ |
| Multi-platform (web + native) | | ✓ |
| Complex transaction logic | ✓ | |
| 3DS card auth needed | ✓ (with returnUrl) | ✓ (built-in) |

### When to use quote-based vs external transactions

| Scenario | Quote-based | External |
|----------|------------|----------|
| User initiates withdrawal | ✓ | |
| User initiates trade | ✓ | |
| User initiates card/APM deposit | ✓ | |
| Crypto sent to deposit address | | ✓ |
| Bank transfer received | | ✓ |
| User pulls funds from external account | ✓ | |

### When to use Uphold-verified vs partner-verified KYC

| Process | Uphold-verified | Partner-verified |
|---------|-----------------|------------------|
| Email, Phone | | ✓ |
| Identity, Proof of address | ✓ | ✓ |
| Customer/Enhanced due diligence | ✓ | ✓ |
| Tax details | ✓ | |
| Crypto risk assessment (UK) | ✓ | ✓ |

### When to use KYC Connector vs direct API

| Scenario | KYC Connector | Direct API |
|----------|---------------|-----------|
| Using Sumsub/Veriff provider | ✓ | |
| Custom KYC UI | | ✓ |
| Embedded KYC Widget | | ✓ |
| Third-party KYC ingestion | ✓ | |

## Workflow

### 1. Onboard a user (individual)
1. **Fetch Terms of Service**: `GET /core/terms-of-service?country={countryCode}`
2. **Create user**: `POST /core/users` with email, phone, profile, country, and accepted ToS
3. **Monitor KYC status**: Poll `GET /core/kyc/{userId}/overview` or subscribe to `core.kyc.*.status-changed` webhooks
4. **Submit KYC data**: Call `PATCH /core/kyc/{userId}/{process}` for each required process (identity, address, CDD, etc.)
5. **Verify capabilities**: `GET /core/users/{userId}/capabilities` to confirm user can transact
6. **Handle periodic review**: Watch for KYC status returning to `pending` (e.g., identity expiring); re-collect and resubmit

### 2. Execute a quote-based transaction (e.g., withdrawal)
1. **Create quote**: `POST /core/transactions/quotes` with origin (account), destination (external account or crypto address), and denomination
2. **Check requirements**: If `requirements` array is non-empty, handle travel rule, 3DS auth, or PayPal auth
3. **Present quote to user**: Show exchange rate, fees, and settlement time
4. **Refresh quote**: Before expiry, call the same endpoint again to get updated rates
5. **Create transaction**: `POST /core/transactions` with quote ID and any required fulfillments (travel rule data, auth token, etc.)
6. **Monitor status**: Subscribe to `core.transaction.status-changed` or poll `GET /core/transactions/{id}`
7. **Handle on-hold**: If status is `on-hold` with RFI reason, fetch pending RFIs via `GET /core/transactions/{id}/rfis` and submit via `PATCH /core/transactions/{id}/rfis/{rfiId}`

### 3. Implement deposit flow with Payment Widget
1. **Create session**: `POST /widgets/payment/sessions` with user ID, flow type (`select-for-deposit`), and optional payment method filters
2. **Initialize widget**: Pass session to `PaymentWidget` SDK in your frontend
3. **Handle events**: Listen for `success` (transaction created), `error` (handle error code), `cancel` (user exited)
4. **Monitor transaction**: Subscribe to `core.transaction.created` and `core.transaction.status-changed` webhooks
5. **Confirm settlement**: When status is `completed`, funds are settled in user's account

### 4. Link external account (bank or card)
1. **Create external account**: `POST /core/external-accounts` with account details (IBAN, card number, etc.)
2. **Monitor status**: Subscribe to `core.external-account.status-changed` or poll `GET /core/external-accounts/{id}`
3. **Handle verification**: Some accounts require microdeposit verification; platform will create disbursement transactions automatically
4. **Use in transactions**: Reference external account ID as destination in withdrawal quotes or origin in pull deposits

### 5. Set up crypto deposit address
1. **Set deposit method**: `POST /core/accounts/{accountId}/deposit-method` with network and asset
2. **Retrieve address**: Response includes `address` and `tag` (if applicable)
3. **Share with user**: Display address for user to send crypto
4. **Monitor deposit**: Subscribe to `core.transaction.created` for incoming crypto transactions
5. **Handle on-chain confirmation**: Transaction status progresses as blockchain confirms

## Common gotchas

- **OAuth2 token expiry**: Tokens expire; implement refresh logic or request new tokens before expiry
- **Quote expiry**: Quotes are valid for a limited time (typically minutes); refresh before expiry or user sees stale rates
- **Missing capabilities**: User may lack required capability (e.g., `crypto-deposits`) even if KYC is complete; always check `GET /core/users/{id}/capabilities` before offering transaction type
- **Unsettled funds**: ACH and some bank deposits don't settle immediately; user cannot withdraw locked funds until settlement; check `transaction_amount_invalid` error with `transacting-unsettled-funds` rule
- **KYC periodic review**: KYC status can revert to `pending` (e.g., identity expiring); monitor webhooks and prompt user to re-verify
- **Transaction on-hold with RFI**: Transaction may be paused pending requests for information; fetch and submit RFIs to resume
- **Rate limits**: Implement exponential backoff on 429 responses; respect `Reset-After` header
- **Webhook signature verification**: Always verify webhook signatures using Svix SDK or manual verification to prevent spoofing
- **Subject context**: Ensure correct subject (client vs user) in requests; organization-wide clients need `X-On-Behalf-Of` header
- **Denomination vs asset**: Denomination is the price reference currency, not necessarily the asset being transacted; e.g., buy 1 BTC priced in USD but pay in EUR
- **External account node types**: Bank external accounts are `external-account` type; crypto addresses are `crypto-address` type; don't confuse them
- **Crypto execution modes**: Crypto transactions can be `onchain`, `offchain`, or `simulated`; check `destination.node.execution.mode` in response
- **CDD for crypto**: All users (including US users) require Customer Due Diligence before crypto deposits/withdrawals; don't skip for US users
- **Postman workspace**: Use the public Postman workspace for testing; fork it to your account and set credentials in variables drawer, not Environments tab

## Verification checklist

Before submitting work:

- [ ] OAuth2 token obtained and valid (check expiry)
- [ ] Correct subject context used (client or user:individual/user:business)
- [ ] Organization-wide clients include `X-On-Behalf-Of` header when acting on behalf of user
- [ ] All required KYC processes completed and status is `ok` (not `pending`, `running`, or `failed`)
- [ ] User capabilities verified for intended transaction type (deposits, trades, withdrawals, etc.)
- [ ] Quote created and presented to user before transaction execution
- [ ] Quote refreshed if approaching expiry
- [ ] All transaction requirements handled (travel rule, 3DS auth, PayPal auth)
- [ ] Webhook subscriptions configured for monitoring (user, KYC, account, transaction events)
- [ ] Webhook signatures verified on receipt
- [ ] Error responses checked for specific error codes (not just HTTP status)
- [ ] Rate limit headers respected; backoff implemented for 429 responses
- [ ] Transaction status monitored via webhooks or polling until completion
- [ ] On-hold transactions with RFIs detected and RFIs submitted
- [ ] External accounts verified before use in transactions
- [ ] Crypto deposit addresses generated and shared with user
- [ ] Payment Widget session created with correct flow type and payment method filters
- [ ] Widget event handlers (success, error, cancel) implemented
- [ ] Denomination currency understood and correct for transaction context
- [ ] Unsettled funds checked before allowing withdrawal
- [ ] Postman workspace used for initial testing (Sandbox environment)

## Resources

- **Comprehensive page navigation**: https://developer.uphold.com/llms.txt
- **Core API concepts and data model**: https://developer.uphold.com/rest-apis/core-api/concepts
- **REST API introduction and OpenAPI specs**: https://developer.uphold.com/rest-apis/introduction
- **Authentication and OAuth2**: https://developer.uphold.com/rest-apis/authentication
- **Webhooks and event subscription**: https://developer.uphold.com/rest-apis/webhooks
- **Transactions API (quotes, RFQ model, RFIs)**: https://developer.uphold.com/rest-apis/core-api/transactions/introduction
- **User onboarding and KYC workflows**: https://developer.uphold.com/developer-guides/user-onboarding/individual/overview
- **Payment Widget for deposits/withdrawals**: https://developer.uphold.com/widgets/payment/introduction
- **Error handling and status codes**: https://developer.uphold.com/rest-apis/errors
- **Rate limits and backoff**: https://developer.uphold.com/rest-apis/rate-limits
- **Postman workspace**: https://www.postman.com/uphold-enterprise/workspace/uphold-enterprise-api/overview (fork to your account)

---

> For additional documentation and navigation, see: https://developer.uphold.com/llms.txt