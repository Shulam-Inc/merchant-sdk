# Merchant SDK Project Plan

## Overview

Server-side middleware that lets any API accept x402 payments with one line of code. Returns HTTP 402 with payment details, validates X-PAYMENT headers, and calls the Shulam facilitator to verify and settle.

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

// Types
import type {
  PaymentRequiredResponse,
  PaymentHeader,
  SettlementResult,
  WebhookPayload,
} from "@shulam/merchant-sdk";
```
