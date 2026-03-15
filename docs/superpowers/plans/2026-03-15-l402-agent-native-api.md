# L402 Agent-Native API Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add L402 Lightning-native authentication and token-based billing to the Bitcoin Companies API, enabling AI agents to register, pay, and consume paid endpoints without any website interaction.

**Architecture:** Simplified L402 (preimage-based, no macaroons). Agents pay a Lightning invoice to register, receive an API key with 10,000 tokens. Paid endpoints deduct tokens. When tokens run out, agents top up via Lightning. Uses existing Phoenixd integration and polymorphic Payment model.

**Tech Stack:** Rails 8.1, SQLite3, Phoenixd (Lightning), RSpec + Rswag

**Spec:** `docs/superpowers/specs/2026-03-15-l402-agent-native-api-design.md`

---

## File Structure

### New files
| File | Responsibility |
|------|---------------|
| `db/migrate/XXXX_create_api_keys.rb` | Migration for api_keys table |
| `app/models/api_key.rb` | Key generation, hashing, token management |
| `app/serializers/api/v1/api_key_serializer.rb` | JSON formatting for API key responses |
| `app/controllers/api/v1/keys_controller.rb` | Registration, balance, topup endpoints |
| `app/controllers/concerns/l402_authentication.rb` | Middleware concern for token gating |
| `app/services/l402_invoice_service.rb` | Invoice creation for registration + topup |
| `spec/requests/api/v1/keys_spec.rb` | RSpec + Rswag tests for keys endpoints |
| `spec/requests/api/v1/l402_authentication_spec.rb` | Integration tests for token gating |

### Modified files
| File | Changes |
|------|---------|
| `app/controllers/api/v1/base_controller.rb` | Include L402Authentication concern, update CORS |
| `app/controllers/api/v1/addresses_controller.rb` | Mark as paid endpoint |
| `app/controllers/api/v1/transactions_controller.rb` | Mark as paid endpoint |
| `app/controllers/api/v1/claims_controller.rb` | Mark as paid endpoint |
| `app/controllers/api/v1/reviews_controller.rb` | Mark as paid endpoint, add create action |
| `config/routes.rb` | Add keys routes |
| `spec/swagger_helper.rb` | Add api_key + L402 error schemas |

---

## Chunk 1: Data Model + ApiKey Model

### Task 1: Create api_keys migration

**Files:**
- Create: `db/migrate/XXXX_create_api_keys.rb`

- [ ] **Step 1: Generate the migration**

Run:
```bash
/opt/homebrew/bin/mise exec -- bin/rails generate migration CreateApiKeys
```

- [ ] **Step 2: Write the migration**

Edit the generated file:

```ruby
class CreateApiKeys < ActiveRecord::Migration[8.1]
  def change
    create_table :api_keys do |t|
      t.string :key_digest, null: false
      t.string :key_prefix, null: false
      t.integer :token_balance, default: 0, null: false
      t.integer :total_tokens_purchased, default: 0, null: false
      t.datetime :last_used_at
      t.timestamps
    end

    add_index :api_keys, :key_digest, unique: true
    add_index :api_keys, :key_prefix
  end
end
```

- [ ] **Step 3: Run the migration**

Run: `/opt/homebrew/bin/mise exec -- bin/rails db:migrate`
Expected: Migration succeeds, `api_keys` table created.

- [ ] **Step 4: Commit**

```bash
git add db/migrate/*_create_api_keys.rb db/schema.rb
git commit -m "feat: add api_keys table for L402 token billing"
```

---

### Task 2: ApiKey model

**Files:**
- Create: `app/models/api_key.rb`

- [ ] **Step 1: Write the model**

```ruby
# frozen_string_literal: true

class ApiKey < ApplicationRecord
  REGISTRATION_SATS = 1_000
  REGISTRATION_TOKENS = 10_000
  KEY_PREFIX_LENGTH = 15

  TOPUP_PACKS = [
    { sats: 5_000, tokens: 5_000 },
    { sats: 25_000, tokens: 25_000 },
    { sats: 100_000, tokens: 100_000 }
  ].freeze

  has_many :payments, as: :payable, dependent: :destroy

  validates :key_digest, presence: true, uniqueness: true
  validates :key_prefix, presence: true
  validates :token_balance, numericality: { greater_than_or_equal_to: 0 }

  # Generate a new API key. Returns the plain key (only shown once).
  def self.create_with_key!(tokens: REGISTRATION_TOKENS)
    plain_key = generate_key
    api_key = create!(
      key_digest: digest(plain_key),
      key_prefix: plain_key[0, KEY_PREFIX_LENGTH],
      token_balance: tokens,
      total_tokens_purchased: tokens
    )
    [api_key, plain_key]
  end

  # Find an ApiKey by plain key (from Authorization header).
  def self.find_by_key(plain_key)
    return nil if plain_key.blank?
    find_by(key_digest: digest(plain_key))
  end

  # Deduct tokens atomically. Returns true if deducted, false if insufficient.
  def deduct_tokens!(cost)
    rows = self.class.where(id: id)
      .where("token_balance >= ?", cost)
      .update_all(["token_balance = token_balance - ?", cost])
    if rows > 0
      reload
      true
    else
      false
    end
  end

  # Add tokens after a topup payment.
  def add_tokens!(amount)
    self.class.where(id: id).update_all(
      ["token_balance = token_balance + ?, total_tokens_purchased = total_tokens_purchased + ?", amount, amount]
    )
    reload
  end

  def touch_last_used!
    update_column(:last_used_at, Time.current)
  end

  def self.generate_key
    "sk_live_#{SecureRandom.hex(24)}"
  end

  def self.digest(plain_key)
    Digest::SHA256.hexdigest(plain_key)
  end
end
```

- [ ] **Step 2: Verify in Rails console**

Run:
```bash
/opt/homebrew/bin/mise exec -- bin/rails runner "
  api_key, plain_key = ApiKey.create_with_key!
  puts \"Key: #{plain_key}\"
  puts \"Prefix: #{api_key.key_prefix}\"
  puts \"Balance: #{api_key.token_balance}\"
  found = ApiKey.find_by_key(plain_key)
  puts \"Found: #{found.id == api_key.id}\"
  puts \"Deduct 1: #{api_key.deduct_tokens!(1)}\"
  puts \"Balance after: #{api_key.token_balance}\"
"
```

Expected: Key created, found by digest, tokens deducted.

- [ ] **Step 3: Commit**

```bash
git add app/models/api_key.rb
git commit -m "feat: add ApiKey model with key generation and token management"
```

---

### Task 3: ApiKey serializer

**Files:**
- Create: `app/serializers/api/v1/api_key_serializer.rb`

- [ ] **Step 1: Write the serializer**

```ruby
# frozen_string_literal: true

module Api
  module V1
    class ApiKeySerializer
      def initialize(api_key, include_key: false, plain_key: nil)
        @api_key = api_key
        @include_key = include_key
        @plain_key = plain_key
      end

      def as_json
        result = {
          id: "key_#{@api_key.id}",
          object: "api_key",
          key_prefix: @api_key.key_prefix,
          token_balance: @api_key.token_balance,
          total_tokens_purchased: @api_key.total_tokens_purchased,
          last_used_at: @api_key.last_used_at&.iso8601,
          created_at: @api_key.created_at&.iso8601
        }

        result[:key] = @plain_key if @include_key && @plain_key

        result
      end
    end
  end
end
```

- [ ] **Step 2: Commit**

```bash
git add app/serializers/api/v1/api_key_serializer.rb
git commit -m "feat: add ApiKeySerializer for L402 API responses"
```

---

## Chunk 2: L402 Invoice Service + Keys Controller

### Task 4: L402 invoice service

**Files:**
- Create: `app/services/l402_invoice_service.rb`

- [ ] **Step 1: Write the service**

```ruby
# frozen_string_literal: true

class L402InvoiceService
  attr_reader :payable, :amount_sats, :memo, :plain_key

  def initialize(payable:, amount_sats:, memo:, plain_key: nil)
    @payable = payable
    @amount_sats = amount_sats
    @memo = memo
    @plain_key = plain_key
  end

  def create_invoice!
    invoice = phoenixd_client.create_invoice(
      amount_sats: amount_sats,
      memo: memo
    )

    meta = {
      payment_request: invoice[:payment_request],
      expires_at: 1.hour.from_now.iso8601
    }
    # Store plain key temporarily so it can be returned after payment verification.
    # The key is only exposed once in the 201 response, then this metadata is no longer needed.
    meta[:plain_key] = plain_key if plain_key

    payment = payable.payments.create!(
      invoice_id: invoice[:payment_hash],
      sats: amount_sats,
      state: "pending",
      provider: "phoenixd",
      metadata: meta
    )

    {
      invoice: invoice[:payment_request],
      payment_hash: invoice[:payment_hash],
      amount_sats: amount_sats,
      payment: payment
    }
  end

  # Verify a preimage against a pending payment.
  # Returns the Payment if valid, nil otherwise.
  #
  # Encoding contract: both preimage and payment_hash are hex strings.
  # Lightning preimage is 32 raw bytes. SHA-256(raw_preimage_bytes) == payment_hash_hex.
  # Agent sends preimage as hex, Phoenixd stores payment_hash as hex.
  def self.verify_preimage(payment_hash:, preimage:)
    return nil if payment_hash.blank? || preimage.blank?

    # Cryptographic verification: SHA-256(preimage_bytes) must equal payment_hash
    # [preimage].pack("H*") converts hex string to raw bytes, hexdigest returns hex
    computed_hash = Digest::SHA256.hexdigest([preimage].pack("H*"))
    return nil unless ActiveSupport::SecurityUtils.secure_compare(computed_hash, payment_hash)

    payment = Payment.find_by(invoice_id: payment_hash, state: "pending")
    return nil unless payment

    payment.update!(state: "paid")

    # Background reconciliation with Phoenixd (non-blocking)
    VerifyPhoenixdPaymentJob.perform_later(payment_hash) if defined?(VerifyPhoenixdPaymentJob)

    payment
  end

  private

  def phoenixd_client
    @phoenixd_client ||= Client::PhoenixdApi.new
  end
end
```

- [ ] **Step 2: Commit**

```bash
git add app/services/l402_invoice_service.rb
git commit -m "feat: add L402InvoiceService for Lightning invoice creation and preimage verification"
```

---

### Task 5: Keys controller

**Files:**
- Create: `app/controllers/api/v1/keys_controller.rb`

- [ ] **Step 1: Write the controller**

```ruby
# frozen_string_literal: true

module Api
  module V1
    class KeysController < BaseController
      # POST /api/v1/keys — Register (L402 flow)
      def create
        # Step 1: Check for L402 payment proof
        payment_hash, preimage = extract_l402_credentials
        if payment_hash && preimage
          payment = L402InvoiceService.verify_preimage(
            payment_hash: payment_hash,
            preimage: preimage
          )

          if payment&.payable.is_a?(ApiKey)
            api_key = payment.payable
            # Grant registration tokens (idempotent via token check)
            if api_key.token_balance == 0 && api_key.total_tokens_purchased == 0
              api_key.add_tokens!(ApiKey::REGISTRATION_TOKENS)
            end
            # Recover the plain key stored in payment metadata during invoice creation
            plain_key = payment.metadata["plain_key"]
            return render json: ApiKeySerializer.new(api_key, include_key: true, plain_key: plain_key).as_json,
                          status: :created
          end

          return render json: { error: { type: "invalid_request_error", message: "Invalid or expired payment proof" } },
                        status: :bad_request
        end

        # Step 2: No payment proof — create invoice
        api_key, plain_key = ApiKey.create_with_key!(tokens: 0) # Tokens added after payment

        invoice_data = L402InvoiceService.new(
          payable: api_key,
          amount_sats: ApiKey::REGISTRATION_SATS,
          memo: "Bitcoin Companies API key registration",
          plain_key: plain_key  # Store temporarily so it can be returned after payment
        ).create_invoice!

        response.headers["WWW-Authenticate"] = %(L402 invoice="#{invoice_data[:invoice]}", hash="#{invoice_data[:payment_hash]}")

        render json: {
          error: {
            type: "payment_required",
            message: "Pay #{ApiKey::REGISTRATION_SATS} sats to register and receive #{ApiKey::REGISTRATION_TOKENS} tokens",
            invoice: invoice_data[:invoice],
            amount_sats: ApiKey::REGISTRATION_SATS,
            payment_hash: invoice_data[:payment_hash],
            tokens_included: ApiKey::REGISTRATION_TOKENS
          }
        }, status: :payment_required

      rescue Client::PhoenixdApi::ApiError => e
        render json: { error: { type: "server_error", message: "Payment service unavailable" } },
              status: :service_unavailable
      end

      # GET /api/v1/keys/me — Check balance
      def show
        api_key = authenticate_api_key!
        return unless api_key

        render json: ApiKeySerializer.new(api_key).as_json
      end

      # POST /api/v1/keys/me/topup — Buy more tokens
      def topup
        api_key = authenticate_api_key!
        return unless api_key

        # Check for payment proof in body
        if params[:payment_hash].present? && params[:preimage].present?
          payment = L402InvoiceService.verify_preimage(
            payment_hash: params[:payment_hash],
            preimage: params[:preimage]
          )

          if payment&.payable == api_key
            pack = ApiKey::TOPUP_PACKS.find { |p| p[:sats] == payment.sats }
            tokens = pack ? pack[:tokens] : payment.sats # Fallback: 1 sat = 1 token
            api_key.add_tokens!(tokens)
            return render json: ApiKeySerializer.new(api_key).as_json
          end

          return render json: { error: { type: "invalid_request_error", message: "Invalid or expired payment proof" } },
                        status: :bad_request
        end

        # No proof — create topup invoice
        sats = params[:sats].to_i
        pack = ApiKey::TOPUP_PACKS.find { |p| p[:sats] == sats }
        unless pack
          return render json: {
            error: {
              type: "invalid_request_error",
              message: "Invalid pack. Choose from: #{ApiKey::TOPUP_PACKS.map { |p| p[:sats] }.join(', ')} sats",
              packs: ApiKey::TOPUP_PACKS
            }
          }, status: :bad_request
        end

        invoice_data = L402InvoiceService.new(
          payable: api_key,
          amount_sats: pack[:sats],
          memo: "Bitcoin Companies API topup: #{pack[:tokens]} tokens"
        ).create_invoice!

        response.headers["WWW-Authenticate"] = %(L402 invoice="#{invoice_data[:invoice]}", hash="#{invoice_data[:payment_hash]}")

        render json: {
          error: {
            type: "payment_required",
            message: "Pay #{pack[:sats]} sats to receive #{pack[:tokens]} tokens",
            invoice: invoice_data[:invoice],
            amount_sats: pack[:sats],
            payment_hash: invoice_data[:payment_hash],
            tokens_included: pack[:tokens]
          }
        }, status: :payment_required

      rescue Client::PhoenixdApi::ApiError => e
        render json: { error: { type: "server_error", message: "Payment service unavailable" } },
              status: :service_unavailable
      end

      private

      def extract_l402_credentials
        auth = request.headers["Authorization"]
        return nil unless auth&.start_with?("L402 ")

        parts = auth.delete_prefix("L402 ").split(":")
        return nil unless parts.length == 2

        [parts[0], parts[1]]
      end

      def authenticate_api_key!
        auth = request.headers["Authorization"]
        unless auth&.start_with?("Bearer ")
          render json: { error: { type: "unauthorized", message: "API key required. Include Authorization: Bearer sk_live_..." } },
                 status: :unauthorized
          return nil
        end

        plain_key = auth.delete_prefix("Bearer ")
        api_key = ApiKey.find_by_key(plain_key)
        unless api_key
          render json: { error: { type: "unauthorized", message: "Invalid API key" } },
                 status: :unauthorized
          return nil
        end

        api_key
      end
    end
  end
end
```

- [ ] **Step 2: Add routes**

Modify `config/routes.rb`. Inside the `namespace :v1` block, add before the catch-all:

```ruby
# L402 API key management
resources :keys, only: [:create] do
  collection do
    get :me, action: :show
    post "me/topup", action: :topup
  end
end
```

- [ ] **Step 3: Commit**

```bash
git add app/controllers/api/v1/keys_controller.rb config/routes.rb
git commit -m "feat: add KeysController with L402 registration and topup flows"
```

---

## Chunk 3: L402 Authentication Middleware

### Task 6: L402Authentication concern

**Files:**
- Create: `app/controllers/concerns/l402_authentication.rb`

- [ ] **Step 1: Write the concern**

```ruby
# frozen_string_literal: true

# Include in BaseController. Free by default.
# Controllers that need token gating override l402_paid_endpoint? to return true.
module L402Authentication
  extend ActiveSupport::Concern

  PAID_COST = 1
  WRITE_COST = 100

  included do
    before_action :check_l402_token
    after_action :set_token_balance_header
  end

  private

  def check_l402_token
    # Always try to resolve the API key (for X-Token-Balance on free endpoints)
    plain_key = extract_bearer_token
    @current_api_key = ApiKey.find_by_key(plain_key) if plain_key

    # Free endpoints: no gating, just resolve key for header
    return unless l402_paid_endpoint?

    unless @current_api_key
      if plain_key
        return render json: { error: { type: "unauthorized", message: "Invalid API key" } },
                      status: :unauthorized
      else
        return render_l402_registration_required
      end
    end

    cost = l402_write_endpoint? ? WRITE_COST : PAID_COST
    unless @current_api_key.deduct_tokens!(cost)
      return render_l402_insufficient_tokens
    end

    @current_api_key.touch_last_used!
  end

  def set_token_balance_header
    response.headers["X-Token-Balance"] = @current_api_key.token_balance.to_s if @current_api_key
  end

  # Override in paid controllers to return true
  def l402_paid_endpoint?
    false
  end

  # Override in controllers with write actions
  def l402_write_endpoint?
    false
  end

  def extract_bearer_token
    auth = request.headers["Authorization"]
    return nil unless auth&.start_with?("Bearer ")
    auth.delete_prefix("Bearer ")
  end

  def render_l402_registration_required
    render json: {
      error: {
        type: "payment_required",
        message: "API key required. Register at POST /api/v1/keys (#{ApiKey::REGISTRATION_SATS} sats, includes #{ApiKey::REGISTRATION_TOKENS} tokens)",
        registration_url: "/api/v1/keys"
      }
    }, status: :payment_required
  end

  def render_l402_insufficient_tokens
    render json: {
      error: {
        type: "insufficient_tokens",
        message: "Token balance is #{@current_api_key.token_balance}. Top up to continue.",
        token_balance: @current_api_key.token_balance,
        topup_url: "/api/v1/keys/me/topup",
        packs: ApiKey::TOPUP_PACKS
      }
    }, status: :payment_required
  end
end
```

- [ ] **Step 2: Commit**

```bash
git add app/controllers/concerns/l402_authentication.rb
git commit -m "feat: add L402Authentication concern for token-gated API endpoints"
```

---

### Task 7: Wire up L402 to paid controllers

**Files:**
- Modify: `app/controllers/api/v1/base_controller.rb`
- Modify: `app/controllers/api/v1/addresses_controller.rb`
- Modify: `app/controllers/api/v1/transactions_controller.rb`
- Modify: `app/controllers/api/v1/claims_controller.rb`
- Modify: `app/controllers/api/v1/reviews_controller.rb`

- [ ] **Step 1: Update BaseController: include L402, update CORS, add OPTIONS**

In `app/controllers/api/v1/base_controller.rb`:

1. Add `include L402Authentication` at the top of the class
2. Update `set_cors_headers`:
```ruby
def set_cors_headers
  response.headers["Access-Control-Allow-Origin"] = "*"
  response.headers["Access-Control-Allow-Methods"] = "GET, POST, OPTIONS"
  response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
end
```
3. Add OPTIONS handler (prevents CORS preflight failures):
```ruby
# In routes.rb, add before the catch-all:
match "/api/v1/*path", to: -> (_) { [204, { "Access-Control-Allow-Origin" => "*", "Access-Control-Allow-Methods" => "GET, POST, OPTIONS", "Access-Control-Allow-Headers" => "Content-Type, Authorization" }, []] }, via: :options
```

Since L402Authentication is included in BaseController with `l402_paid_endpoint?` returning `false` by default, all existing free endpoints remain unaffected.

- [ ] **Step 2: Mark paid controllers**

Each paid controller overrides `l402_paid_endpoint?` to return `true`. No `include` needed (it's in BaseController).

**AddressesController** — add private method:
```ruby
private
def l402_paid_endpoint?
  true
end
```

**TransactionsController** — same pattern.

**ClaimsController** — same pattern. Read the file first to confirm the class structure.

**ReviewsController** — needs paid + write endpoint overrides:
```ruby
private
def l402_paid_endpoint?
  true
end

def l402_write_endpoint?
  action_name == "create"
end
```

Also add a `create` action to ReviewsController:

```ruby
def create
  company = Company.active.find_by!(domain: params[:company_domain])

  review = company.reviews.create!(
    overall_rating: params.require(:overall_rating),
    title: params[:title],
    body: params.require(:body),
    author_name: params[:author_name] || "API User",
    source: Source.find_or_create_by!(name: "api"),
    state: "published",
    published_at: Time.current
  )

  render json: ReviewSerializer.new(review).as_json, status: :created
end
```

- [ ] **Step 3: Add review create route**

In `config/routes.rb`, change `resources :reviews, only: [:index]` to:
```ruby
resources :reviews, only: [:index, :create]
```

- [ ] **Step 4: Verify free endpoints still work without auth**

Run:
```bash
curl -s http://localhost:3000/api/v1/companies?limit=1 | python3 -m json.tool | head -5
curl -s http://localhost:3000/api/v1/price | python3 -m json.tool
```

Expected: Both return 200 OK with data (no auth needed).

- [ ] **Step 5: Verify paid endpoints require auth**

Run:
```bash
curl -s http://localhost:3000/api/v1/companies/strategy.com/addresses | python3 -m json.tool
```

Expected: 402 with `payment_required` error and registration URL.

- [ ] **Step 6: Commit**

```bash
git add app/controllers/api/v1/base_controller.rb \
        app/controllers/api/v1/addresses_controller.rb \
        app/controllers/api/v1/transactions_controller.rb \
        app/controllers/api/v1/claims_controller.rb \
        app/controllers/api/v1/reviews_controller.rb \
        config/routes.rb
git commit -m "feat: wire L402 authentication to paid API endpoints"
```

---

### Task 8: Update existing specs with API key auth

**Files:**
- Modify: `spec/requests/api/v1/addresses_spec.rb`
- Modify: `spec/requests/api/v1/transactions_spec.rb`
- Modify: `spec/requests/api/v1/reviews_spec.rb`
- Modify: `spec/requests/api/v1/claims_spec.rb` (if it exists)

- [ ] **Step 1: Add API key setup to each paid endpoint spec**

Each of these spec files now needs a valid API key since the endpoints are gated. Add to each spec file:

```ruby
# At the top of the RSpec.describe block, add:
let(:api_key_record) { ApiKey.create_with_key! }
let(:api_key) { api_key_record[0] }
let(:plain_key) { api_key_record[1] }

# In each response "200" block, add:
let(:Authorization) { "Bearer #{plain_key}" }
```

Also add `parameter name: :Authorization, in: :header, type: :string, required: true` to each `get` block.

For the 404 test cases, also include the Authorization header (otherwise they'll get 402 instead of 404).

- [ ] **Step 2: Run all existing specs to verify they pass**

Run: `/opt/homebrew/bin/mise exec -- bundle exec rspec spec/requests/api/v1/ --format documentation`
Expected: All existing specs still pass with the new auth headers.

- [ ] **Step 3: Commit**

```bash
git add spec/requests/api/v1/addresses_spec.rb \
        spec/requests/api/v1/transactions_spec.rb \
        spec/requests/api/v1/reviews_spec.rb \
        spec/requests/api/v1/claims_spec.rb
git commit -m "fix: update paid endpoint specs with API key auth headers"
```

---

## Chunk 4: Swagger Schemas + RSpec Tests

### Task 9: Add swagger schemas

**Files:**
- Modify: `spec/swagger_helper.rb`

- [ ] **Step 1: Add api_key and L402 error schemas**

Add to the `schemas` section in `spec/swagger_helper.rb`:

```ruby
api_key: {
  type: :object,
  properties: {
    id: { type: :string, example: "key_1" },
    object: { type: :string, enum: ["api_key"] },
    key_prefix: { type: :string, example: "sk_live_a1b2c3d" },
    key: { type: :string, example: "sk_live_a1b2c3d4e5f6...", description: "Only returned once at creation" },
    token_balance: { type: :integer, example: 10000 },
    total_tokens_purchased: { type: :integer, example: 10000 },
    last_used_at: { type: :string, format: "date-time", nullable: true },
    created_at: { type: :string, format: "date-time" }
  },
  required: %w[id object key_prefix token_balance]
},
l402_error: {
  type: :object,
  properties: {
    error: {
      type: :object,
      properties: {
        type: { type: :string, example: "payment_required" },
        message: { type: :string },
        invoice: { type: :string, description: "BOLT11 Lightning invoice" },
        amount_sats: { type: :integer },
        payment_hash: { type: :string },
        tokens_included: { type: :integer }
      }
    }
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add spec/swagger_helper.rb
git commit -m "feat: add api_key and l402_error swagger schemas"
```

---

### Task 10: Keys endpoint RSpec tests

**Files:**
- Create: `spec/requests/api/v1/keys_spec.rb`

- [ ] **Step 1: Write the spec**

```ruby
require "swagger_helper"

RSpec.describe "API Keys" do
  path "/api/v1/keys" do
    post "Register for an API key" do
      tags "Keys"
      description "L402 flow: first call returns 402 with Lightning invoice. Pay, then retry with L402 Authorization header to receive API key."
      produces "application/json"

      response "402", "payment required (first call)" do
        schema "$ref" => "#/components/schemas/l402_error"

        before do
          # Stub Phoenixd to avoid real Lightning calls in tests
          allow_any_instance_of(Client::PhoenixdApi).to receive(:create_invoice).and_return(
            { payment_hash: "test_hash_abc", payment_request: "lnbc10u1ptest..." }
          )
        end

        run_test! do |response|
          json = JSON.parse(response.body)
          expect(json["error"]["type"]).to eq("payment_required")
          expect(json["error"]["invoice"]).to be_present
          expect(json["error"]["amount_sats"]).to eq(1000)
          expect(json["error"]["tokens_included"]).to eq(10000)
          expect(response.headers["WWW-Authenticate"]).to include("L402")
        end
      end
    end
  end

  path "/api/v1/keys/me" do
    get "Check API key balance" do
      tags "Keys"
      description "Returns current token balance and key metadata."
      produces "application/json"
      security [{ bearer: [] }]

      parameter name: :Authorization, in: :header, type: :string, required: true,
                description: "Bearer sk_live_..."

      response "200", "balance returned" do
        let(:api_key_record) { ApiKey.create_with_key! }
        let(:Authorization) { "Bearer #{api_key_record[1]}" }

        schema "$ref" => "#/components/schemas/api_key"

        run_test! do |response|
          json = JSON.parse(response.body)
          expect(json["object"]).to eq("api_key")
          expect(json["token_balance"]).to eq(10000)
        end
      end

      response "401", "invalid API key" do
        let(:Authorization) { "Bearer sk_live_invalid" }

        schema "$ref" => "#/components/schemas/error"

        run_test! do |response|
          json = JSON.parse(response.body)
          expect(json["error"]["type"]).to eq("unauthorized")
        end
      end
    end
  end
end
```

- [ ] **Step 2: Run the specs**

Run: `/opt/homebrew/bin/mise exec -- bundle exec rspec spec/requests/api/v1/keys_spec.rb -v`
Expected: All tests pass.

- [ ] **Step 3: Commit**

```bash
git add spec/requests/api/v1/keys_spec.rb
git commit -m "test: add RSpec tests for L402 keys endpoints"
```

---

### Task 11: L402 authentication integration tests

**Files:**
- Create: `spec/requests/api/v1/l402_authentication_spec.rb`

- [ ] **Step 1: Write the spec**

```ruby
require "rails_helper"

RSpec.describe "L402 Authentication", type: :request do
  let!(:api_key_record) { ApiKey.create_with_key! }
  let(:api_key) { api_key_record[0] }
  let(:plain_key) { api_key_record[1] }
  let(:test_domain) { Company.active.joins(:addresses).first&.domain || "strategy.com" }

  describe "free endpoints" do
    it "works without auth" do
      get "/api/v1/companies"
      expect(response).to have_http_status(:ok)
    end

    it "includes X-Token-Balance when auth is present" do
      get "/api/v1/companies", headers: { "Authorization" => "Bearer #{plain_key}" }
      expect(response).to have_http_status(:ok)
      expect(response.headers["X-Token-Balance"]).to eq("10000")
    end
  end

  describe "paid endpoints" do
    it "returns 402 without auth" do
      get "/api/v1/companies/#{test_domain}/addresses"
      expect(response).to have_http_status(:payment_required)
      json = JSON.parse(response.body)
      expect(json["error"]["type"]).to eq("payment_required")
      expect(json["error"]["registration_url"]).to eq("/api/v1/keys")
    end

    it "returns 401 with invalid key" do
      get "/api/v1/companies/#{test_domain}/addresses",
          headers: { "Authorization" => "Bearer sk_live_invalid" }
      expect(response).to have_http_status(:unauthorized)
    end

    it "deducts 1 token and returns data" do
      get "/api/v1/companies/#{test_domain}/addresses",
          headers: { "Authorization" => "Bearer #{plain_key}" }
      expect(response).to have_http_status(:ok)
      expect(response.headers["X-Token-Balance"]).to eq("9999")
      expect(api_key.reload.token_balance).to eq(9999)
    end

    it "returns 402 when tokens exhausted" do
      api_key.update!(token_balance: 0)
      get "/api/v1/companies/#{test_domain}/addresses",
          headers: { "Authorization" => "Bearer #{plain_key}" }
      expect(response).to have_http_status(:payment_required)
      json = JSON.parse(response.body)
      expect(json["error"]["type"]).to eq("insufficient_tokens")
      expect(json["error"]["packs"]).to be_an(Array)
    end
  end
end
```

- [ ] **Step 2: Run the specs**

Run: `/opt/homebrew/bin/mise exec -- bundle exec rspec spec/requests/api/v1/l402_authentication_spec.rb -v`
Expected: All tests pass.

- [ ] **Step 3: Run ALL API specs to ensure no regressions**

Run: `/opt/homebrew/bin/mise exec -- bundle exec rspec spec/requests/api/v1/ --format documentation`
Expected: All specs pass (existing free endpoints unaffected).

- [ ] **Step 4: Regenerate swagger docs**

Run: `/opt/homebrew/bin/mise exec -- bundle exec rake api:docs`
Expected: Swagger regenerated with new schemas.

- [ ] **Step 5: Commit**

```bash
git add spec/requests/api/v1/l402_authentication_spec.rb swagger/v1/swagger.yaml
git commit -m "test: add L402 authentication integration tests"
```

---

## Chunk 5: Update Existing Specs + Root Endpoint + Skill

### Task 12: Update root endpoint with new routes

**Files:**
- Modify: `app/controllers/api/v1/root_controller.rb`

- [ ] **Step 1: Add keys and paid endpoint info to root response**

Add to the `endpoints` hash:

```ruby
keys: "/api/v1/keys",
keys_balance: "/api/v1/keys/me",
keys_topup: "/api/v1/keys/me/topup",
addresses: "/api/v1/companies/:domain/addresses",
transactions: "/api/v1/companies/:domain/transactions",
reviews: "/api/v1/companies/:domain/reviews"
```

- [ ] **Step 2: Commit**

```bash
git add app/controllers/api/v1/root_controller.rb
git commit -m "feat: add L402 key endpoints to API root directory"
```

---

### Task 13: Update SKILL.md with L402 handling

**Files:**
- Modify: `skills/btc/SKILL.md` (in bitcoin-companies-skills repo)

- [ ] **Step 1: Add API key instructions**

Add a new section after the API Reference in SKILL.md:

```markdown
## Authentication

Some endpoints require an API key. If a request returns `402 Payment Required`:

1. Check if `BITCOINCOMPANIES_API_KEY` is set in the environment
2. If set, include it: `Authorization: Bearer $BITCOINCOMPANIES_API_KEY`
3. If not set, display: "This endpoint requires an API key. Get one at POST /api/v1/keys (1,000 sats, includes 10,000 tokens). Set BITCOINCOMPANIES_API_KEY in your environment."

**On all WebFetch calls**, if `BITCOINCOMPANIES_API_KEY` env var exists, pass it as a Bearer token.

**Paid endpoints** (1 token/call): addresses, transactions, claims, reviews (GET)
**Write endpoints** (100 tokens): reviews (POST)
**Free endpoints** (no key needed): companies, price, tiers, stats, countries, weather, map
```

- [ ] **Step 2: Commit** (in skills repo)

```bash
cd /Users/sbounmy/code/bitcoin-companies-skills
git add skills/btc/SKILL.md
git commit -m "feat: add L402 API key authentication handling to skill"
```

---

### Task 14: Final integration test

- [ ] **Step 1: Run full test suite**

```bash
/opt/homebrew/bin/mise exec -- bundle exec rspec spec/requests/api/v1/ --format documentation
/opt/homebrew/bin/mise exec -- bundle exec rails test
```

Expected: All specs pass.

- [ ] **Step 2: Manual smoke test**

```bash
# Free endpoint (no auth)
curl -s http://localhost:3000/api/v1/companies?limit=1 | python3 -m json.tool | head -3

# Paid endpoint without auth (expect 402)
curl -s -w "\n%{http_code}" http://localhost:3000/api/v1/companies/strategy.com/addresses

# Create API key in console, test with it
/opt/homebrew/bin/mise exec -- bin/rails runner "
  key, plain = ApiKey.create_with_key!
  puts plain
"

# Use the key
curl -s -H 'Authorization: Bearer sk_live_FROM_ABOVE' \
  http://localhost:3000/api/v1/companies/strategy.com/addresses?limit=1 | python3 -m json.tool | head -5

# Check balance
curl -s -H 'Authorization: Bearer sk_live_FROM_ABOVE' \
  http://localhost:3000/api/v1/keys/me | python3 -m json.tool
```

- [ ] **Step 3: Regenerate API docs**

```bash
/opt/homebrew/bin/mise exec -- bundle exec rake api:docs
```

- [ ] **Step 4: Final commit**

```bash
git add swagger/v1/swagger.yaml
git commit -m "docs: regenerate OpenAPI docs with L402 endpoints"
```
