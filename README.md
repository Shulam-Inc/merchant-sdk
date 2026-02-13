# @shulam/merchant-sdk

Server-side middleware for accepting x402 payments. One line of code to monetize any API.

## Quick Start

```bash
npm install @shulam/merchant-sdk
```

```typescript
import express from "express";
import { paymentRequired } from "@shulam/merchant-sdk";

const app = express();

app.get("/premium-data",
  paymentRequired({ amount: "0.10", address: "0xYourWallet..." }),
  (req, res) => {
    // Only runs after valid payment + settlement
    console.log(req.x402); // { txHash, fee, netAmount, ... }
    res.json({ data: "valuable" });
  }
);
```

## How It Works

1. Client requests your API without payment → middleware returns **HTTP 402** with pricing
2. Client signs an EIP-3009 authorization and retries with `X-PAYMENT` header
3. Middleware verifies the payment via the Shulam facilitator
4. Middleware settles the payment on-chain (USDC on Base)
5. Your route handler runs with `req.x402` containing the settlement result

## Features

- **Express middleware** — `paymentRequired(options)`
- **Next.js support** — `withPayment(handler, options)`
- **Webhook verification** — `verifyWebhookSignature(payload, sig, secret)`
- **Dynamic pricing** — `amount: (req) => calculatePrice(req)`
- **TypeScript first** — full type definitions for all x402 types

## Framework Support

| Framework | Import | Status |
|-----------|--------|--------|
| Express | `@shulam/merchant-sdk` | Planned |
| Next.js | `@shulam/merchant-sdk/nextjs` | Planned |
| Fastify | `@shulam/merchant-sdk/fastify` | Planned |
| Core (any) | `@shulam/merchant-sdk/core` | Planned |
