# Merchant SDK Project Plan

## Overview

Server-side middleware that lets any API accept x402 payments with one line of code. Returns HTTP 402 with payment details, validates X-PAYMENT headers, and calls the Shulam facilitator to verify and settle.

Also provides a **project scaffolder** (`npx @shulam/merchant-sdk init`) and an **x402 capability manifest** (`x402-manifest.json`) so that AI agents and automated clients can discover what endpoints cost, what currencies are accepted, and what SLAs are offered — enabling agent-to-agent commerce.

**Package name:** `@shulam/merchant-sdk`

```typescript
import { paymentRequired } from "@shulam/merchant-sdk";

app.get("/premium-data", paymentRequired({ amount: "0.10", address: "0x..." }), (req, res) => {
  res.json({ data: "valuable" });
});
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     MERCHANT SDK                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────┐        │
│  │              Framework Adapters                  │        │
│  │                                                  │        │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │        │
│  │  │ Express  │ │ Next.js  │ │ Fastify  │        │        │
│  │  │Middleware│ │Middleware│ │  Plugin  │        │        │
│  │  └──────────┘ └──────────┘ └──────────┘        │        │
│  └─────────────────────────────────────────────────┘        │
│                          │                                   │
│  ┌─────────────────────────────────────────────────┐        │
│  │              Core Library                        │        │
│  │                                                  │        │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │        │
│  │  │  402     │ │ Payment  │ │ Webhook  │        │        │
│  │  │ Response │ │ Verify   │ │ Verify   │        │        │
│  │  │ Builder  │ │ Client   │ │ Helper   │        │        │
│  │  └──────────┘ └──────────┘ └──────────┘        │        │
│  └─────────────────────────────────────────────────┘        │
│                          │                                   │
│                          ▼                                   │
│               ┌──────────────────┐                          │
│               │   Facilitator    │                          │
│               │   API Client     │                          │
│               │  /verify /settle │                          │
│               └──────────────────┘                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| facilitator | Internal | /verify and /settle API calls |
| viem | External | Address validation, type helpers |
| zod | External | Config validation |

---

## Milestones

### M1: Core Library
- [ ] 402 response builder (payment details JSON per x402 spec)
- [ ] X-PAYMENT header parser (base64url decode)
- [ ] Facilitator API client (verify + settle)
- [ ] Config schema (Zod): amount, address, network, facilitatorUrl
- [ ] TypeScript types for x402 payment request/response

### M2: Express Middleware
- [ ] `paymentRequired(options)` middleware factory
- [ ] Returns 402 with payment details when no X-PAYMENT header
- [ ] Validates payment header, calls /verify, calls /settle
- [ ] Attaches settlement result to `req.x402` for downstream handlers
- [ ] Error handling (invalid payment, settlement failure)

### M3: Next.js Middleware
- [ ] `withPayment(handler, options)` wrapper for API routes
- [ ] App Router middleware support
- [ ] Edge runtime compatible

### M4: Webhook Verification
- [ ] `verifyWebhookSignature(payload, signature, secret)` helper
- [ ] Express middleware: `webhookVerifier(secret)`
- [ ] TypeScript types for all webhook event payloads

### M5: Advanced Features
- [ ] Per-route pricing (different amounts per endpoint)
- [ ] Dynamic pricing callback (`amount: (req) => calculatePrice(req)`)
- [ ] Rate limiting integration (x402 as rate limit bypass)
- [ ] Request metering (track usage per buyer address)
- [ ] Multi-chain support (network selection per route)

### M6: Fastify Plugin
- [ ] `fastifyPaymentRequired` plugin
- [ ] Matches Express middleware API surface

### M7: Project Scaffolder
- [ ] `npx @shulam/merchant-sdk init <name>` CLI command
- [ ] Generates Express app with `paymentRequired()` middleware pre-wired
- [ ] Includes testkit as devDependency with mock facilitator setup
- [ ] Webhook endpoint stub with signature verification
- [ ] `.env.example` with facilitator URL, webhook secret, wallet address
- [ ] Template selection: Express, Next.js, or Fastify
- [ ] `npm test` runs against mock facilitator out of the box

### M8: x402 Capability Manifest
- [ ] `x402-manifest.json` spec — machine-readable endpoint pricing and capabilities
- [ ] Manifest schema (Zod): endpoints, pricing, accepted currencies, SLAs, rate limits
- [ ] `GET /.well-known/x402-manifest.json` auto-served by middleware
- [ ] `generateManifest(app)` — introspect Express/Fastify routes to build manifest
- [ ] Manifest fields: endpoint URL, method, amount, currency, description, rateLimit, ttl
- [ ] Versioned manifest (`"version": "1.0"`) for forward compatibility
- [ ] Manifest validation CLI: `npx @shulam/merchant-sdk validate-manifest`

---

## User Stories (Gherkin)

### Epic 1: One-Line Integration

```gherkin
Feature: Payment Middleware
  As a merchant developer
  I want to add payment requirements with one line of code
  So that I can monetize my API instantly

  Scenario: Return 402 when no payment attached
    Given my Express app has the paymentRequired middleware on /api/data
    And the middleware is configured with amount "0.10" and my wallet address
    When a client sends GET /api/data without an X-PAYMENT header
    Then the server returns HTTP 402
    And the response body contains:
      | field              | value                |
      | maxAmountRequired  | 0.10                 |
      | payTo              | 0x<my-address>       |
      | asset              | USDC contract address|
      | network            | base-sepolia         |

  Scenario: Accept valid payment and serve resource
    Given my Express app has the paymentRequired middleware on /api/data
    When a client sends GET /api/data with a valid X-PAYMENT header
    Then the middleware calls POST /verify on the facilitator
    And the middleware calls POST /settle on the facilitator
    And the request proceeds to the route handler
    And req.x402 contains the settlement result (txHash, fee, netAmount)

  Scenario: Reject invalid payment
    Given my Express app has the paymentRequired middleware on /api/data
    When a client sends GET /api/data with an invalid X-PAYMENT header
    Then the middleware calls POST /verify on the facilitator
    And the facilitator returns verified: false
    Then the server returns HTTP 402 with error details
    And the route handler is NOT called
```

### Epic 2: Webhook Verification

```gherkin
Feature: Webhook Signature Verification
  As a merchant developer
  I want to verify webhook signatures
  So that I can trust incoming payment notifications

  Scenario: Verify valid webhook
    Given I have my webhook secret from the facilitator
    When I receive a POST with X-Shulam-Signature header
    And I call verifyWebhookSignature(body, signature, secret)
    Then it returns true

  Scenario: Reject tampered webhook
    Given I have my webhook secret
    When I receive a POST with a modified body
    And the signature does not match
    Then verifyWebhookSignature returns false
```

### Epic 3: Project Scaffolder

```gherkin
Feature: Merchant Project Scaffolder
  As a merchant developer
  I want to scaffold a new project with payments pre-configured
  So that I can start accepting payments in minutes

  Scenario: Scaffold Express project
    When I run "npx @shulam/merchant-sdk init my-store"
    Then a directory "my-store" is created
    And it contains an Express app with paymentRequired middleware on /api/data
    And package.json includes @shulam/merchant-sdk and @shulam/testkit
    And .env.example contains SHULAM_FACILITATOR_URL and SHULAM_WEBHOOK_SECRET
    And "npm test" passes against the mock facilitator

  Scenario: Scaffold Next.js project
    When I run "npx @shulam/merchant-sdk init my-store --template nextjs"
    Then a Next.js app is created with withPayment middleware on /api/data
    And the middleware returns 402 for unauthenticated requests

  Scenario: Scaffold with custom wallet address
    When I run "npx @shulam/merchant-sdk init my-store --address 0xABC..."
    Then .env contains SHULAM_MERCHANT_ADDRESS=0xABC...
    And the paymentRequired middleware uses that address
```

### Epic 4: x402 Capability Manifest

```gherkin
Feature: x402 Capability Manifest
  As an AI agent or automated client
  I want to discover what endpoints cost and what they offer
  So that I can decide whether to pay before making requests

  Scenario: Serve manifest at well-known URL
    Given my Express app uses @shulam/merchant-sdk middleware on 3 routes
    When a client sends GET /.well-known/x402-manifest.json
    Then the response is a JSON manifest with:
      """
      {
        "version": "1.0",
        "provider": "My API",
        "endpoints": [
          {
            "path": "/api/data",
            "method": "GET",
            "amount": "0.10",
            "currency": "USDC",
            "network": "base",
            "description": "Premium dataset access",
            "rateLimit": "100/hour"
          }
        ]
      }
      """

  Scenario: Auto-generate manifest from routes
    Given my Express app has paymentRequired middleware on multiple routes
    When I call generateManifest(app)
    Then a manifest JSON is returned with all paid endpoints
    And each entry includes the amount and description from middleware config

  Scenario: Agent discovers pricing before paying
    Given an AI agent wants to access /api/data
    When the agent fetches /.well-known/x402-manifest.json
    Then the agent sees the endpoint costs 0.10 USDC
    And the agent compares this against its budget
    And the agent decides whether to proceed

  Scenario: Validate manifest via CLI
    Given I have an x402-manifest.json file
    When I run "npx @shulam/merchant-sdk validate-manifest"
    Then the CLI validates the manifest against the Zod schema
    And reports any errors or warnings
```

---

## Package Structure

```
merchant-sdk/
├── src/
│   ├── core/
│   │   ├── types.ts              # x402 payment types
│   │   ├── response-builder.ts   # Build 402 response body
│   │   ├── header-parser.ts      # Parse X-PAYMENT header
│   │   └── facilitator-client.ts # HTTP client for /verify, /settle
│   ├── express/
│   │   ├── middleware.ts         # paymentRequired() factory
│   │   └── webhook.ts           # webhookVerifier() middleware
│   ├── nextjs/
│   │   ├── with-payment.ts      # withPayment() wrapper
│   │   └── middleware.ts         # Edge middleware
│   ├── fastify/
│   │   └── plugin.ts            # Fastify plugin
│   ├── manifest/
│   │   ├── types.ts              # x402ManifestSchema (Zod)
│   │   ├── generator.ts          # generateManifest(app) introspection
│   │   ├── middleware.ts         # Serve /.well-known/x402-manifest.json
│   │   └── validator.ts          # CLI manifest validation
│   ├── cli/
│   │   ├── init.ts               # Project scaffolder
│   │   ├── validate-manifest.ts  # Manifest validation command
│   │   └── index.ts              # CLI entry point
│   ├── templates/
│   │   ├── express/              # Express starter template
│   │   ├── nextjs/               # Next.js starter template
│   │   └── fastify/              # Fastify starter template
│   └── index.ts                  # Main barrel export
├── tests/
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

---

## Environment Variables

```bash
# Required
SHULAM_FACILITATOR_URL=https://api.shulam.io  # or http://localhost:3000
SHULAM_WEBHOOK_SECRET=                         # from webhook registration

# Optional
SHULAM_DEFAULT_NETWORK=base-sepolia
SHULAM_DEFAULT_ASSET=USDC
```

---

## API Surface

```typescript
// Express middleware
import { paymentRequired, webhookVerifier } from "@shulam/merchant-sdk";

// Next.js
import { withPayment } from "@shulam/merchant-sdk/nextjs";

// Core (framework-agnostic)
import {
  buildPaymentResponse,
  parsePaymentHeader,
  FacilitatorClient,
  verifyWebhookSignature,
} from "@shulam/merchant-sdk/core";

// Manifest (agent-to-agent discovery)
import {
  generateManifest,
  serveManifest,
  validateManifest,
} from "@shulam/merchant-sdk/manifest";

// Types
import type {
  PaymentRequiredResponse,
  PaymentHeader,
  SettlementResult,
  WebhookPayload,
  X402Manifest,
  ManifestEndpoint,
} from "@shulam/merchant-sdk";
```

## CLI

```bash
# Scaffold a new merchant project
npx @shulam/merchant-sdk init my-store
npx @shulam/merchant-sdk init my-store --template nextjs
npx @shulam/merchant-sdk init my-store --address 0xABC...

# Validate an x402 manifest
npx @shulam/merchant-sdk validate-manifest
npx @shulam/merchant-sdk validate-manifest ./custom-manifest.json
```
