# L402 Agent-Native API Design

## Goal

Add Lightning-native authentication and token-based billing to the Bitcoin Companies API, enabling AI agents to register, pay, and consume paid endpoints without any website interaction.

## Context

The Bitcoin Companies API (`/api/v1`) is currently free and public. This design adds:
- **L402 payment flow**: Agents pay Lightning invoices to register and top up
- **Token-based billing**: Paid endpoints deduct tokens (1 token = 1 sat)
- **Agent-native onboarding**: No email, no password, no website. Just Lightning.

The ecosystem is ready: [Alby MCP Server](https://github.com/getAlby/mcp) connects AI agents to Lightning wallets via Nostr Wallet Connect. Agents with Alby MCP can pay L402 invoices autonomously. We just need to return standard L402 responses.

## Terminology

| Term | Definition |
|------|-----------|
| **API key** | `sk_live_...` string that authenticates the agent |
| **Tokens** | Spendable balance, 1 token = 1 sat |
| **L402** | Lightning Labs HTTP 402 protocol for payment-gated APIs. We implement a simplified variant (preimage-based, no macaroons) for MVP. See L402 Header Format section. |

## Architecture

```
Agent/Client
    |
    v
[L402 Middleware] -- checks API key + token balance, returns 402 if needed
    |
    v
[API Endpoints] -- same controllers as today
    |
    v
[Phoenixd] -- generates BOLT11 invoices, verifies preimage payments
```

### Three agent states

| State | Free endpoints | Paid endpoints | Write endpoints |
|-------|---------------|----------------|-----------------|
| No API key | Works (60 req/min) | 402 → register | 402 → register |
| API key + tokens | Works (60 req/min) | Deducts 1 token | Deducts 100 tokens |
| API key + 0 tokens | Works (60 req/min) | 402 → topup | 402 → topup |

### Endpoint classification

**Free (no API key needed):**
- `GET /api/v1/companies` (leaderboard)
- `GET /api/v1/companies/:domain` (company detail)
- `GET /api/v1/price`
- `GET /api/v1/tiers`
- `GET /api/v1/stats`
- `GET /api/v1/countries`
- `GET /api/v1/weather`
- `GET /api/v1/companies/:domain/weather`

**Paid (API key + 1 token/call):**
- `GET /api/v1/companies/:domain/addresses`
- `GET /api/v1/companies/:domain/transactions`
- `GET /api/v1/companies/:domain/claims`
- `GET /api/v1/companies/:domain/reviews`

**Write (API key + 100 tokens):**
- `POST /api/v1/companies/:domain/reviews`

**Key management (L402):**
- `POST /api/v1/keys` — register (1,000 sats → 10,000 tokens)
- `GET /api/v1/keys/me` — check balance (free, requires API key)
- `POST /api/v1/keys/me/topup` — buy more tokens

## L402 Flows

### Registration (get an API key)

```
1. Agent:  POST /api/v1/keys
2. Server: 402 Payment Required
           WWW-Authenticate: L402 invoice="lnbc10u1p...", hash="abc123"
           {
             "error": {
               "type": "payment_required",
               "message": "Pay 1,000 sats to register and receive 10,000 tokens",
               "invoice": "lnbc10u1p...",
               "amount_sats": 1000,
               "payment_hash": "abc123",
               "tokens_included": 10000
             }
           }

3. Agent:  Pays BOLT11 invoice (via Alby MCP, NWC wallet, or manually)
           Gets preimage back from wallet

4. Agent:  POST /api/v1/keys
           Authorization: L402 abc123:preimage_hex

5. Server: 201 Created
           {
             "id": "key_abc123",
             "object": "api_key",
             "key": "sk_live_a1b2c3d4e5f6...",
             "token_balance": 10000,
             "created_at": "2026-03-15T12:00:00Z"
           }
```

The `key` field is only returned once at creation. Server stores a SHA-256 hash.

### Using paid endpoints

```
1. Agent:  GET /api/v1/companies/strategy.com/addresses
           Authorization: Bearer sk_live_a1b2c3d4e5f6...

2. Server: 200 OK
           X-Token-Balance: 9999
           { "object": "list", "data": [...] }
```

### Out of tokens

```
1. Agent:  GET /api/v1/companies/strategy.com/addresses
           Authorization: Bearer sk_live_a1b2c3d4e5f6...

2. Server: 402 Payment Required
           {
             "error": {
               "type": "insufficient_tokens",
               "message": "Token balance is 0. Top up to continue.",
               "token_balance": 0,
               "topup_url": "/api/v1/keys/me/topup",
               "packs": [
                 { "sats": 5000, "tokens": 5000 },
                 { "sats": 25000, "tokens": 25000 },
                 { "sats": 100000, "tokens": 100000 }
               ]
             }
           }
```

### Top up

```
1. Agent:  POST /api/v1/keys/me/topup
           Authorization: Bearer sk_live_a1b2c3d4e5f6...
           Body: { "sats": 5000 }

2. Server: 402 Payment Required
           WWW-Authenticate: L402 invoice="lnbc50u1p...", hash="def456"
           {
             "error": {
               "type": "payment_required",
               "message": "Pay 5,000 sats to receive 5,000 tokens",
               "invoice": "lnbc50u1p...",
               "amount_sats": 5000,
               "payment_hash": "def456",
               "tokens_included": 5000
             }
           }

3. Agent:  Pays invoice, gets preimage

4. Agent:  POST /api/v1/keys/me/topup
           Authorization: Bearer sk_live_a1b2c3d4e5f6...
           Body: { "sats": 5000, "payment_hash": "def456", "preimage": "preimage_hex" }

5. Server: 200 OK
           {
             "id": "key_abc123",
             "object": "api_key",
             "token_balance": 5000,
             "total_tokens_purchased": 15000
           }
```

### Posting a review (write endpoint)

```
1. Agent:  POST /api/v1/companies/binance.com/reviews
           Authorization: Bearer sk_live_a1b2c3d4e5f6...
           Body: { "title": "Solid exchange", "body": "Fast withdrawals...", "overall_rating": 4 }

2. Server: 201 Created (deducts 100 tokens)
           X-Token-Balance: 4900
           { "id": 42, "object": "review", "state": "published", ... }
```

### Check balance

```
1. Agent:  GET /api/v1/keys/me
           Authorization: Bearer sk_live_a1b2c3d4e5f6...

2. Server: 200 OK
           {
             "id": "key_abc123",
             "object": "api_key",
             "key_prefix": "sk_live_a1b2...",
             "token_balance": 4900,
             "total_tokens_purchased": 15000,
             "last_used_at": "2026-03-15T14:30:00Z",
             "created_at": "2026-03-15T12:00:00Z"
           }
```

## Pricing

| Pack | Price | Tokens | Note |
|------|-------|--------|------|
| Registration | 1,000 sats (~$1) | 10,000 | One-time welcome bonus |
| Refill | 5,000 sats (~$5) | 5,000 | 1 sat = 1 token |
| Builder | 25,000 sats (~$25) | 25,000 | 1 sat = 1 token |
| Pro | 100,000 sats (~$100) | 100,000 | 1 sat = 1 token |
| Enterprise | Contact | Custom | Custom |

Base rate for top-ups is 1 sat = 1 token. Registration is a one-time exception: pay 1,000 sats, get 10,000 tokens (includes a 9,000 token welcome bonus to let agents try everything).

### Endpoint costs

| Endpoint type | Cost |
|--------------|------|
| Free (leaderboard, price, stats) | 0 tokens |
| Paid read (addresses, transactions, claims, reviews) | 1 token |
| Write (post review) | 100 tokens |

## Data Model

### New: ApiKey

```ruby
create_table :api_keys do |t|
  t.string :key_digest, null: false, index: { unique: true }  # SHA-256 of sk_live_...
  t.string :key_prefix, null: false                             # "sk_live_a1b2" for display
  t.integer :token_balance, default: 0, null: false
  t.integer :total_tokens_purchased, default: 0, null: false
  t.datetime :last_used_at
  t.timestamps
end
```

### Existing: Payment (polymorphic)

```ruby
# Already supports payable_type: "Review"
# Add: payable_type: "ApiKey" for registration + topup payments
# Fields: sats, state ("pending"/"paid"/"expired"), provider ("phoenixd"),
#         invoice_id (payment_hash), metadata (includes BOLT11, expires_at)
```

### Invoice expiry

BOLT11 invoices have a default expiry (typically 1 hour for Phoenixd). Handle expired invoices:
- Store `expires_at` in Payment metadata when creating the invoice
- If agent sends a preimage for an expired Payment, verify the preimage anyway (payment may have settled before expiry)
- If preimage is invalid AND invoice expired, return 402 with a fresh invoice
- Background job cleans up stale `pending` payments older than 24 hours (marks them `expired`)

### Key security

- Full API key (`sk_live_...`) shown only once at creation
- Server stores SHA-256 digest for lookup: `Digest::SHA256.hexdigest(key)`
- `key_prefix` stored for identification: first 15 characters, e.g. `sk_live_a1b2c3d`
- Lookup: hash the incoming Bearer token, find by digest

## Middleware: L402 Authentication

New concern `L402Authentication` added to `Api::V1::BaseController`:

```ruby
# before_action :check_l402, only: paid + write endpoints

# Pseudocode:
# 1. Skip if free endpoint
# 2. Extract Bearer key from Authorization header
# 3. If no key and endpoint is paid → 402 with registration invoice
# 4. If key invalid → 401 Unauthorized
# 5. If key has insufficient tokens → 402 with topup packs
# 6. Deduct tokens (1 for reads, 100 for writes)
# 7. Set X-Token-Balance response header (on ALL responses when API key is present, including free endpoints)
```

Token deduction must be atomic to prevent race conditions:

```ruby
ApiKey.where(id: key.id)
      .where("token_balance >= ?", cost)
      .update_all(["token_balance = token_balance - ?", cost])
```

## L402 Header Format

Simplified L402 (preimage-based, no macaroons). The full [Lightning Labs L402 spec](https://docs.lightning.engineering/the-lightning-network/l402) uses macaroons with caveats. We use a simplified variant for MVP since most AI agent wallets (Alby MCP, NWC) work with raw preimages. Can upgrade to full macaroon-based L402 later if interop with `lnget` or Aperture is needed.

**Server → Agent (402 response):**
```
HTTP/1.1 402 Payment Required
WWW-Authenticate: L402 invoice="lnbc...", hash="payment_hash_hex"
Content-Type: application/json
```

**Agent → Server (payment proof on registration):**
```
Authorization: L402 <payment_hash>:<preimage_hex>
```

**Agent → Server (payment proof on topup, already authenticated):**
```
Authorization: Bearer sk_live_...
Body: { "sats": 5000, "payment_hash": "def456", "preimage": "preimage_hex" }
```

**Server verification:**
- Compute `SHA-256(preimage)` and check it matches the `payment_hash`. This is cryptographic proof of payment and is sufficient on its own.
- Background reconciliation: also verify with Phoenixd `check_payment(hash)` asynchronously. If the preimage is valid, the payment happened. Phoenixd check is belt-and-suspenders, not a blocking gate.
- Mark associated Payment record as paid

**Idempotency:** The `payment_hash` serves as the idempotency key. If an agent retries with the same valid preimage (e.g., network dropped the response), return the previously created API key (registration) or confirm the existing top-up. The Payment model's `invoice_id` field (which stores the payment hash) enforces uniqueness.

## Agent Wallet Integration (Not Our Responsibility)

Agents pay L402 invoices using their own wallet. We just return standard L402 responses. Existing ecosystem options:

| Solution | How it works |
|----------|-------------|
| [Alby MCP](https://github.com/getAlby/mcp) | MCP server for Claude Code/Cursor. User adds NWC connection string, agent pays autonomously. |
| [NWC](https://nwc.dev/) | Nostr Wallet Connect. Any NWC-compatible wallet. Budget controls built in. |
| [Lightning Agent Tools](https://lightning.engineering/posts/2026-03-11-L402-for-agents/) | Lightning Labs toolkit: lnd + lnget + aperture for full L402 agent stack. |
| Manual | Agent shows BOLT11 to user, user pays with any wallet, agent retries. |

Setup for Claude Code users:
```bash
claude mcp add alby -- npx -y @getalby/mcp
# Set NWC connection string from Alby Hub
```

## Multi-Rail Support (Future)

The 402 response is designed to support multiple payment rails later:

```json
{
  "error": {
    "type": "payment_required",
    "payment_options": [
      { "rail": "lightning", "invoice": "lnbc...", "amount": "1000 sats" },
      { "rail": "usdc_base", "address": "0x...", "amount": "0.01 USDC" }
    ]
  }
}
```

Server-side: extract verification into a strategy pattern:
- `LightningVerifier` → Phoenixd `check_payment(hash)`
- `StablecoinVerifier` → check USDC transfer on Base (future)

The `Payment` model already has a `provider` field to distinguish rails.

**Not built now.** Lightning only for MVP. Architecture supports adding x402/USDC later without breaking changes.

## Plugin Skill Updates

Update `/btc` skill to handle L402:

1. When a paid endpoint returns 402, display: "This endpoint requires an API key. Register at POST /api/v1/keys (1,000 sats) or set BITCOINCOMPANIES_API_KEY in your environment."
2. If `BITCOINCOMPANIES_API_KEY` env var is set, include it as `Authorization: Bearer` on all requests.
3. If agent has Alby MCP, it can pay the L402 invoice autonomously.

## Security Considerations

- API keys are hashed (SHA-256) before storage. Plain key shown once.
- Token deduction is atomic (SQL `UPDATE ... WHERE balance >= cost`).
- L402 preimage verification is cryptographic (SHA-256 hash match).
- Belt-and-suspenders: also verify with Phoenixd `check_payment`.
- Rate limiting: unauthenticated requests limited by IP (60 req/min), authenticated requests limited by API key (60 req/min). Authenticated requests do not count against IP limits.
- CORS: update to allow `GET, POST, OPTIONS` methods and `Content-Type, Authorization` headers. Add OPTIONS preflight handler.
- No email, no PII stored. API key is pseudonymous.
- API-created reviews skip the Phoenixd QR code payment flow and go straight to `published` state. This is acceptable because the tokens used to post were already paid for with real sats during registration or top-up. The Lightning payment happened at the token level, not the review level.

## Files to Create/Modify

### New files
- `app/models/api_key.rb` — model with key generation, hashing, token management
- `app/controllers/api/v1/keys_controller.rb` — registration, balance, topup
- `app/serializers/api/v1/api_key_serializer.rb` — JSON formatting
- `app/controllers/concerns/l402_authentication.rb` — middleware concern
- `db/migrate/XXXX_create_api_keys.rb` — migration
- `spec/requests/api/v1/keys_spec.rb` — RSpec tests
- `spec/requests/api/v1/l402_spec.rb` — L402 flow integration tests

### Modified files
- `app/controllers/api/v1/base_controller.rb` — add L402 middleware
- `app/controllers/api/v1/addresses_controller.rb` — mark as paid
- `app/controllers/api/v1/transactions_controller.rb` — mark as paid
- `app/controllers/api/v1/reviews_controller.rb` — mark as paid, add create action
- `app/controllers/api/v1/claims_controller.rb` — mark as paid
- `config/routes.rb` — add keys routes, update CORS
- `spec/swagger_helper.rb` — add api_key schema

### Skills repo
- `skills/btc/SKILL.md` — add API key handling, L402 error display
