# geo-price

Location-based regional pricing for web applications.

`geo-price` lets you show and charge country-specific prices without exposing pricing rules or payment amounts to the browser. Prices are configured per country in minor currency units. The server owns the pricing decision; the client only displays the result and never supplies an amount or currency for checkout.

This is a pricing SDK, not a currency converter. Regional prices are intentionally independent per market and are **not** generated from live exchange rates.

npm packages: [`@geo-price/client`](https://www.npmjs.com/package/@geo-price/client) and [`@geo-price/server`](https://www.npmjs.com/package/@geo-price/server)

## Packages

| Package | Environment | Purpose |
| --- | --- | --- |
| [`@geo-price/client`](https://www.npmjs.com/package/@geo-price/client) | Browser | Geolocation, session setup, price display |
| [`@geo-price/server`](https://www.npmjs.com/package/@geo-price/server) | Node.js | Sessions, pricing, public price data, payment orders |

Both packages ship TypeScript declarations. There is no shared runtime package and no mandatory framework dependency.

## Requirements

- Node.js 18 or newer for `@geo-price/server`.
- A modern browser with the Geolocation API and `fetch` for `@geo-price/client`.
- HTTPS for browser geolocation in production. Browsers generally allow geolocation on `localhost` during development.
- Your backend must expose three small endpoints that the client calls. The contract is documented in [Required backend endpoints](#required-backend-endpoints).

## Installation

```bash
npm install @geo-price/client @geo-price/server
```

Install only the package you need if you are integrating one side independently:

```bash
npm install @geo-price/client
npm install @geo-price/server
```

Both packages are ESM-only. Use `import`, not `require`.

## How it works

```text
Browser                         Your backend                         Payment provider
   |                                 |                                      |
   |-- navigator.geolocation ------->|                                      |
   |-- POST /geo/resolve ----------->| resolve coordinates to country       |
   |<-------- { countryCode } -------|                                      |
   |-- POST /session --------------->| createPricingSession()               |
   |<---- Set-Cookie + sessionId ----|                                      |
   |-- GET /prices/public ---------->| lookupPrice() per product            |
   |<-------- { product: price } ----|                                      |
   |                                 |                                      |
   |-- POST /checkout/:productId --->| createPaymentOrder() --------------->|
   |<-------- order details ---------|                                      |
   |                                 |<--------- provider webhook ----------|
   |                                 | verifyPayment() + grant access       |
```

1. The client obtains coordinates with the browser Geolocation API. Coordinates are sent only to your backend.
2. Your backend resolves the coordinates to an ISO 3166-1 alpha-2 country code.
3. The client creates a server-managed pricing session. The session id is stored in an opaque cookie.
4. The client fetches the public price list for the resolved country and displays prices.
5. Checkout is created on the server from the session's country and your server-side pricing configuration.
6. Payment verification compares the provider's amount and currency with the stored order.

## Quick start

### 1. Define prices on the server

```ts
// pricing.ts
import { definePricing } from "@geo-price/server";

export const pricing = definePricing({
  premium: {
    US: { amount: 599, currency: "USD" },     // $5.99
    GB: { amount: 479, currency: "GBP" },     // £4.79
    DE: { amount: 499, currency: "EUR" },     // €4.99
    IN: { amount: 17979, currency: "INR" },   // ₹179.79
    JP: { amount: 899, currency: "JPY" },     // ¥899
  },
  basic: {
    US: { amount: 299, currency: "USD" },
    IN: { amount: 9900, currency: "INR" },
  },
});
```

Amounts are integer minor currency units: `599` means `5.99` in a two-decimal currency, `899` means `899` in a zero-decimal currency. See [Currency precision](#currency-precision).

### 2. Add the backend routes

The following Express example implements everything the client needs plus a checkout flow.

```ts
// server.ts
import express from "express";
import cookieParser from "cookie-parser";
import {
  MemorySessionStore,
  calculatePrice,
  createPaymentOrder,
  createPricingSession,
  definePricing,
  lookupPrice,
  resolveCountryByCoordinates,
  sessionCookie,
  verifyPayment,
  type PaymentProvider,
} from "@geo-price/server";

const app = express();
app.use(express.json());
app.use(cookieParser());

const pricing = definePricing({
  premium: {
    US: { amount: 599, currency: "USD" },
    GB: { amount: 479, currency: "GBP" },
    DE: { amount: 499, currency: "EUR" },
    IN: { amount: 17979, currency: "INR" },
    JP: { amount: 899, currency: "JPY" },
  },
});

const store = new MemorySessionStore();

app.post("/geo/resolve", (req, res) => {
  const { latitude, longitude } = req.body ?? {};

  if (typeof latitude !== "number" || typeof longitude !== "number") {
    return res.status(400).json({ error: "latitude and longitude are required" });
  }

  const resolved = resolveCountryByCoordinates(latitude, longitude);
  if (!resolved) {
    return res.status(422).json({ error: "Unable to resolve country" });
  }

  // Optional: reject open-water or very distant matches.
  if (resolved.distanceKm > 50) {
    return res.status(422).json({ error: "Location is too far from a known border" });
  }

  res.json({ countryCode: resolved.code });
});

app.post("/session", async (req, res) => {
  const countryCode = String(req.body?.countryCode ?? "").toUpperCase();

  if (!/^[A-Z]{2}$/.test(countryCode)) {
    return res.status(400).json({ error: "Invalid country code" });
  }

  // Never accept "verified" from the client. The client cannot prove residency.
  const session = await createPricingSession(
    {
      countryCode,
      evidence: "detected",
      ttlMs: 24 * 60 * 60 * 1000,
    },
    store,
  );

  res.setHeader(
    "Set-Cookie",
    sessionCookie(session, {
      secure: process.env.NODE_ENV === "production",
      sameSite: "lax",
    }),
  );
  res.json({ sessionId: session.sessionId });
});

app.get("/prices/public", (req, res) => {
  const countryCode = String(req.query.countryCode ?? "").toUpperCase();

  if (!/^[A-Z]{2}$/.test(countryCode)) {
    return res.status(400).json({ error: "Invalid country code" });
  }

  const prices: Record<string, { amount: number; currency: string }> = {};

  for (const productId of Object.keys(pricing)) {
    const price = lookupPrice(pricing, productId, countryCode);
    if (price) {
      prices[productId] = { amount: price.amount, currency: price.currencyCode };
    }
  }

  res.setHeader("Cache-Control", "public, max-age=300");
  res.json(prices);
});

// Optional: useful for server-rendered pages or your own frontend.
app.get("/price/:productId", async (req, res) => {
  const price = await calculatePrice({
    request: { cookies: req.cookies },
    productId: req.params.productId,
    config: pricing,
    store,
  });

  if (!price) {
    return res.status(404).json({ error: "No session or price" });
  }

  res.json(price);
});

app.listen(4000);
```

`MemorySessionStore` is for development and single-process deployments only. See [Production session stores](#production-session-stores).

### 3. Initialize the client

```ts
// client.ts
import { initGeoPrice } from "@geo-price/client";

export const geoPrice = await initGeoPrice({
  apiUrl: "https://api.example.com",
  detection: "browser-geolocation",
  fallback: {
    mode: "default-country",
    countryCode: "US",
  },
});
```

`initGeoPrice` resolves after initial detection and session setup. It can reject if your backend is unreachable or if the configured fallback cannot establish a session.

### 4. Display a price

```ts
const price = await geoPrice.getPrice("premium");

if (price) {
  console.log(`${price.major} ${price.currencyCode}`);
}
```

In React:

```tsx
import { useEffect, useState } from "react";
import type { Price } from "@geo-price/client";
import { geoPrice } from "./client";

export function PriceTag({ productId }: { productId: string }) {
  const [price, setPrice] = useState<Price | null>(null);

  useEffect(() => {
    let active = true;

    geoPrice.getPrice(productId).then((result) => {
      if (active) setPrice(result);
    });

    return () => {
      active = false;
    };
  }, [productId]);

  if (!price) return <span>Price unavailable</span>;

  return (
    <span>
      {price.major} {price.currencyCode}
    </span>
  );
}
```

### 5. Create checkout on the server

Never accept an amount or currency from the browser.

```ts
type StoredOrder = {
  userId: string;
  amount: number;
  currency: string;
};

const orders = new Map<string, StoredOrder>();

const provider: PaymentProvider = {
  async createOrder({ amount, currency, productId }) {
    // Replace with the real provider SDK call.
    const created = await paymentProvider.createOrder({
      amount,
      currency,
      receipt: productId,
    });

    return { orderId: created.id, clientSecret: created.clientSecret };
  },

  async verifyPayment({ orderId, providerResponse }) {
    // Verify the provider signature or API response here.
    // Return the amount and currency reported by the provider, in minor units.
    return paymentProvider.verify(orderId, providerResponse);
  },
};

app.post("/checkout/:productId", async (req, res) => {
  const order = await createPaymentOrder({
    request: { cookies: req.cookies },
    productId: req.params.productId,
    config: pricing,
    store,
    provider,
    requireEvidence: "verified",
  });

  if (!order) {
    return res.status(403).json({ error: "Not eligible for checkout" });
  }

  // Persist this in a database in production, keyed by orderId.
  orders.set(order.orderId, {
    userId: req.user.id,
    amount: order.amount,
    currency: order.currency,
  });

  res.json(order);
});
```

`requireEvidence: "verified"` rejects sessions whose evidence is only `detected`. The client always creates `detected` sessions. Upgrade a session only after an independent check, such as a billing country or payment-provider country:

```ts
await store.update(session.sessionId, { evidence: "verified" });
```

### 6. Verify payment

```ts
app.post("/payments/verify", async (req, res) => {
  const stored = orders.get(req.body?.orderId);

  if (!stored) {
    return res.status(404).json({ error: "Unknown order" });
  }

  const result = await verifyPayment({
    orderId: req.body.orderId,
    providerResponse: req.body,
    provider,
    expectedAmount: stored.amount,
    expectedCurrency: stored.currency,
  });

  if (result.paid) {
    await grantAccess(stored.userId);
    orders.delete(req.body.orderId);
  }

  res.json(result);
});
```

`expectedAmount` and `expectedCurrency` must come from the stored order, never from the request body. In production, persist orders in a database, make verification idempotent, and verify provider webhook signatures before calling `verifyPayment`.

## Required backend endpoints

The client calls these endpoints directly. They are part of the integration contract.

| Endpoint | Request | Response | Credentials |
| --- | --- | --- | --- |
| `POST /geo/resolve` | `{ latitude: number, longitude: number }` | `{ countryCode: string }` | Not sent |
| `POST /session` | `{ countryCode: string, evidence: "detected" }` | `{ sessionId: string }` and `Set-Cookie` | Included |
| `GET /prices/public?countryCode=XX` | Query string | `Record<productId, { amount: number, currency: string }>` | Included |

Notes:

- The client does not call `/price/:productId`. That endpoint is optional and useful for server-rendered pages or your own frontend code.
- `/session` must set the cookie returned by `sessionCookie()`. The cookie name defaults to `gp_session`.
- `/prices/public` must return prices from your server-side configuration. It is a display API, not a checkout authority.
- Validate and normalize `countryCode` on the server. Do not trust it for checkout; checkout reads the session cookie.
- Every endpoint must return JSON on success. The client treats a non-2xx response or invalid JSON as an error.

## Pricing

### Explicit per-country pricing

```ts
import { definePricing } from "@geo-price/server";

export const pricing = definePricing({
  premium: {
    US: { amount: 599, currency: "USD" },
    IN: { amount: 17979, currency: "INR" },
    JP: { amount: 899, currency: "JPY" },
  },
});
```

`definePricing` returns the configuration unchanged. It provides compile-time shape checking; validate at startup that every amount is a non-negative integer and every currency is a valid ISO 4217 code.

### Currency precision

`amount` is always in minor units.

| Exponent | Currencies |
| --- | --- |
| 0 | BIF, CLP, DJF, GNF, ISK, JPY, KMF, KRW, PYG, RWF, UGX, VND, VUV, XAF, XOF, XPF |
| 2 | Most currencies, including USD, EUR, GBP, INR, CNY, IDR |
| 3 | BHD, IQD, JOD, KWD, LYD, OMR, TND |

IDR uses two minor-unit decimals for storage and display, but derived regional prices are rounded to whole rupiah.

```ts
import { minorUnitExponent, roundingUnitExponent, toMajor } from "@geo-price/server";

minorUnitExponent("JPY");        // 0
minorUnitExponent("USD");        // 2
minorUnitExponent("BHD");        // 3
roundingUnitExponent("IDR");     // 0

toMajor(599, "USD");             // "5.99"
toMajor(899, "JPY");             // "899"
toMajor(9587900, "IDR");         // "95879.00"
toMajor(579, "BHD");             // "0.579"
```

If your payment provider uses a different unit convention, convert in the provider adapter and keep one convention throughout your application. For example, some providers represent JPY in 1/100 units even though ISO 4217 defines zero decimal places.

### Derived regional pricing

For bootstrapping, `defineRegionalPricing` derives prices for the bundled country list from one base price:

```ts
import { defineRegionalPricing } from "@geo-price/server";

const pricing = defineRegionalPricing({
  premium: { amount: 599, currency: "USD" },
});
```

For a $5.99 base price, the defaults produce:

| Market | Minor amount | Display |
| --- | --- | --- |
| United States | `599 USD` | $5.99 |
| United Kingdom | `479 GBP` | £4.79 |
| Germany, France, Switzerland | `499 EUR` / `499 CHF` | €4.99 / CHF 4.99 |
| India | `17979 INR` | ₹179.79 |
| Canada | `799 CAD` | CA$7.99 |
| Australia | `899 AUD` | A$8.99 |
| China | `4299 CNY` | CN¥42.99 |
| Japan | `899 JPY` | ¥899 |
| South Korea | `7789 KRW` | ₩7,789 |
| Indonesia | `9587900 IDR` | Rp95,879 |
| Vietnam | `149779 VND` | ₫149,779 |
| Chile | `5689 CLP` | CLP 5,689 |
| Iceland | `799 ISK` | ISK 799 |
| Burkina Faso, Central African Republic | `3589 XOF` / `3589 XAF` | CFA 3,589 |
| New Caledonia | `679 XPF` | XPF 679 |

The derivation is:

```text
baseMajor      = baseAmount / 10^baseExponent
targetMajor    = baseMajor * rate
roundedMajor   = roundToNearest9(targetMajor * 10^roundingExponent) / 10^roundingExponent
targetAmount   = round(roundedMajor * 10^targetExponent)
```

The `rate` is a local-price multiplier, not an exchange rate. It is applied to the base price in major units. Zero-decimal currencies therefore use whole-unit rates: JPY `150`, KRW `1300`, VND `25000`, XOF/XAF `600`.

> **Important:** The bundled regional ratios are illustrative. They are not live FX rates and are not a substitute for market research. Review every rate or override before production. For production pricing, explicit per-country amounts are recommended.

Options:

```ts
import { REGIONAL_RATIOS, defineRegionalPricing } from "@geo-price/server";

const pricing = defineRegionalPricing(
  { premium: { amount: 599, currency: "USD" } },
  {
    ratios: {
      ...REGIONAL_RATIOS,
      GB: { currency: "GBP", rate: 0.75 },
    },
    defaultRatio: 0.7,
    baseCountry: "US",
    overrides: {
      premium: {
        IN: { amount: 14900, currency: "INR" },
        JP: { amount: 800, currency: "JPY" },
      },
    },
  },
);
```

- `ratios` replaces the bundled ratio table.
- `defaultRatio` is used for countries without an explicit ratio.
- `baseCountry` defaults to `US`.
- `overrides` take precedence over derived prices and are not rounded.
- The base price should be in the base country's currency when using the bundled table. The defaults assume a USD base.

### Charm rounding

`roundToNearest9` rounds a value to the nearest charm ending in the `79`/`89`/`99` cycle. It operates on the number you pass; currency exponents are applied by the caller.

```ts
import { roundToNearest9 } from "@geo-price/server";

roundToNearest9(500);     // 499
roundToNearest9(1000);    // 999
roundToNearest9(4309);    // 4299
roundToNearest9(539.1);   // 499  (opts for the lower price)
roundToNearest9(540);     // 579
```

When two charm candidates are within one minor unit of being equally close, the lower price wins. This avoids an upward jump at the midpoint caused by floating-point noise.

## Client API

### `initGeoPrice(config)`

```ts
import { initGeoPrice } from "@geo-price/client";

const geoPrice = await initGeoPrice({
  apiUrl: "https://api.example.com",
  detection: "browser-geolocation",
  fallback: { mode: "default-country", countryCode: "US" },
});
```

Configuration:

| Option | Type | Required | Description |
| --- | --- | --- | --- |
| `apiUrl` | `string` | Yes | Backend base URL. A trailing slash is ignored. |
| `detection` | `"browser-geolocation"` | Yes | The only built-in detection method. |
| `fallback` | `FallbackPolicy` | Yes | Behavior when geolocation or resolution fails. |
| `countryResolver` | `CountryResolver` | No | Custom coordinate-to-country resolver. When set, `/geo/resolve` is not called. |

A custom resolver must return a country code:

```ts
const geoPrice = await initGeoPrice({
  apiUrl: "https://api.example.com",
  detection: "browser-geolocation",
  countryResolver: {
    async resolve(latitude, longitude) {
      return myResolver(latitude, longitude);
    },
  },
  fallback: { mode: "browse-only" },
});
```

The client does not call a third-party geolocation API. Coordinates go only to the resolver you configure.

### Fallback policies

Exactly one fallback is required. There is no implicit USD default and no unrestricted country selector.

| Mode | Country / evidence | Prices | Server session |
| --- | --- | --- | --- |
| `default-country` | Configured country / `detected` | Fetched from `/prices/public` | Created |
| `default-price` | `null` / `null` | Local fallback prices | None |
| `require-verification` | `null` / `null` | None until your app verifies the user | None |
| `browse-only` | `null` / `null` | None | None |

`require-verification` and `browse-only` behave identically inside the SDK. The difference is how your application gates checkout: require verification, or allow browsing without checkout. In both cases the SDK returns no price until you establish a session through your own verification flow.

`default-price` returns local prices with `Price.countryCode` set to `"fallback"`:

```ts
fallback: {
  mode: "default-price",
  prices: {
    premium: { amount: 599, currency: "USD" },
  },
}
```

`default-country` creates a `detected` session. If checkout requires verified evidence, the order request will be rejected until the session is upgraded.

### Client methods

```ts
interface GeoPriceClient {
  getCountryCode(): Promise<string | null>;
  getEvidenceLevel(): Promise<"detected" | "verified" | null>;
  getPrice(productId: string): Promise<Price | null>;
  getPrices(): Promise<Record<string, Price | null>>;
  refresh(): Promise<void>;
  destroy(): void;
}
```

- `getPrice` reads the cached result from initialization; it does not perform a network request.
- `getPrices` returns every cached product plus any `default-price` entries.
- `refresh` clears the cache and runs detection again. It also creates a new session. The old session remains in the store until it expires.
- `destroy` stops the client and releases the ready promise. It does not delete the server session or clear the cookie.

If geolocation is denied, unavailable, or times out, the client applies the fallback instead of rejecting. `initGeoPrice` can still reject if the fallback's `/session` or `/prices/public` request fails. Handle initialization errors in your application shell.

### Cookies and CORS

The client sends `credentials: "include"` when it calls `/session` and `/prices/public`. For a same-origin deployment, no extra configuration is needed. For a cross-origin API:

- Respond with `Access-Control-Allow-Origin: <your exact frontend origin>` and `Access-Control-Allow-Credentials: true`. A wildcard origin cannot be used with credentials.
- Use `secure: true` on the session cookie.
- Use `sameSite: "none"` when the frontend and API are cross-site. Keep `sameSite: "lax"` when they are same-site but on different origins.
- Serve both sides over HTTPS. `Secure` cookies are not stored over plain HTTP.
- If the API is on a sibling subdomain, consider setting `domain` on the cookie instead of widening `SameSite`.

Local development over HTTP must disable `Secure`:

```ts
sessionCookie(session, {
  secure: false,
  sameSite: "lax",
});
```

The cookie is `HttpOnly` by default, so browser JavaScript cannot read it. Do not try to read the session id in the client.

### Privacy

- Browser geolocation requires explicit user permission.
- The client sends coordinates only to your configured resolver or your own `/geo/resolve` endpoint.
- The SDK does not store coordinates in the session. A session contains the country code, evidence level, and timestamps.
- Do not log raw coordinates longer than necessary; they are precise personal data.
- If permission is denied, the SDK applies your fallback. It does not silently switch to IP-based geolocation.

## Server API

### Sessions

```ts
import {
  MemorySessionStore,
  createPricingSession,
  sessionCookie,
} from "@geo-price/server";

const store = new MemorySessionStore();

const session = await createPricingSession(
  {
    countryCode: "IN",
    evidence: "detected",
    ttlMs: 24 * 60 * 60 * 1000,
  },
  store,
);

res.setHeader("Set-Cookie", sessionCookie(session));
```

A session contains:

```ts
interface PricingSession {
  sessionId: string;
  countryCode: string;
  evidence: "detected" | "verified";
  createdAt: number;
  expiresAt: number;
}
```

Session ids are opaque, prefixed with `gp_`, and generated from a UUID. The default TTL is 24 hours. `MemorySessionStore` removes expired sessions lazily when they are read.

### Cookie options

```ts
sessionCookie(session, {
  name: "gp_session",
  path: "/",
  domain: ".example.com",
  secure: true,
  httpOnly: true,
  sameSite: "lax",
  expires: new Date(session.expiresAt),
});
```

| Option | Default | Notes |
| --- | --- | --- |
| `name` | `gp_session` | Must match `sessionCookieName` when reading prices. |
| `path` | `/` | |
| `domain` | Host-only | Set only when sharing across subdomains. |
| `secure` | `true` | Set `false` for plain-HTTP local development. |
| `httpOnly` | `true` | Keeps the session id away from JavaScript. |
| `sameSite` | `lax` | Use `none` for cross-site deployments; requires `secure`. |
| `expires` | Session expiry | |

To log out, clear the cookie and delete the session:

```ts
import { clearSessionCookie } from "@geo-price/server";

res.setHeader("Set-Cookie", clearSessionCookie());
await store.delete(session.sessionId);
```

### Production session stores

Implement the `SessionStore` interface with a shared store such as Redis, DynamoDB, or PostgreSQL:

```ts
import type { PricingSession, SessionId, SessionStore } from "@geo-price/server";

class RedisSessionStore implements SessionStore {
  async create(session: PricingSession): Promise<PricingSession> {
    // store with a TTL equal to session.expiresAt
    return session;
  }

  async find(sessionId: SessionId): Promise<PricingSession | null> {
    // return null when missing or expired
    return null;
  }

  async update(sessionId: SessionId, patch: Partial<PricingSession>): Promise<void> {
    // merge and preserve the TTL
  }

  async delete(sessionId: SessionId): Promise<void> {
    // remove the key
  }
}
```

`MemorySessionStore` is per process. It is unsuitable for serverless functions, multiple instances, or deployments that restart frequently.

### Calculating a price from the session

```ts
import { calculatePrice } from "@geo-price/server";

const price = await calculatePrice({
  request: { cookies: req.cookies },
  productId: "premium",
  config: pricing,
  store,
});

if (!price) {
  return res.status(404).json({ error: "No session or price" });
}

res.json(price);
```

Response:

```json
{
  "productId": "premium",
  "countryCode": "IN",
  "currencyCode": "INR",
  "amount": 17979,
  "major": "179.79",
  "evidence": "detected",
  "eligible": true
}
```

If the session exists but the product has no price for its country, `calculatePrice` returns `eligible: false` with an empty currency and amount `0`. If there is no session or the session is unknown, it returns `null`.

The client does not call this endpoint. It is useful for server-rendered pages, custom frontends, or diagnostics.

### Public price list

`lookupPrice` returns one product price for a country:

```ts
import { lookupPrice } from "@geo-price/server";

const price = lookupPrice(pricing, "premium", "IN");
// { productId: "premium", countryCode: "IN", currencyCode: "INR", amount: 17979, major: "179.79" }
```

Your `/prices/public` endpoint should return the same `amount` and `currency` values that checkout would use for that country. The query string is a display hint, not proof of residency.

### Checkout orders

```ts
const order = await createPaymentOrder({
  request: { cookies: req.cookies },
  productId: "premium",
  config: pricing,
  store,
  provider,
  requireEvidence: "verified",
});
```

`createPaymentOrder` returns `null` when:

- there is no session or the session is unknown,
- the product has no price for the session's country,
- `requireEvidence: "verified"` is set and the session evidence is `detected`.

Otherwise it calls `provider.createOrder` with the minor-unit amount and currency from your configuration and returns:

```ts
interface PaymentOrder {
  orderId: string;
  amount: number;
  currency: string;
  countryCode: string;
  evidence: "detected" | "verified";
  clientSecret: string | undefined;
}
```

The SDK does not store orders. Persist the order id, amount, currency, country, user, and product before granting access. Use that stored record during verification.

### Payment provider interface

```ts
interface PaymentProvider {
  createOrder(params: {
    amount: number;
    currency: string;
    productId: string;
    metadata?: Record<string, string>;
  }): Promise<{ orderId: string; [key: string]: unknown }>;

  verifyPayment(params: {
    orderId: string;
    providerResponse: unknown;
  }): Promise<{
    paid: boolean;
    amount: number;
    currency: string;
  }>;
}
```

`verifyPayment` must validate the provider's signature or API response and return the amount and currency reported by the provider. Do not trust amount or currency fields sent by the browser.

### Verifying payment

```ts
const result = await verifyPayment({
  orderId,
  providerResponse,
  provider,
  expectedAmount: storedOrder.amount,
  expectedCurrency: storedOrder.currency,
});

if (result.paid) {
  // grant access
}
```

`result` is `{ paid: boolean, orderId: string }`. `paid` is true only when the provider reports success and the amount and currency exactly match the expected values.

### Local country resolution

The server package includes a local resolver backed by Natural Earth 1:50m country polygons. It makes no network calls and needs no API key.

```ts
import { resolveCountryByCoordinates } from "@geo-price/server";

const result = resolveCountryByCoordinates(28.6139, 77.2090);
// {
//   code: "IN",
//   name: "India",
//   currency: "INR",
//   distanceKm: 0,
//   matchedBy: "polygon"
// }
```

`matchedBy` is `"polygon"` when the point is inside a border polygon. If the point falls just outside a simplified coastline or on a small island missing from the data, the resolver uses the nearest border and returns `matchedBy: "border"` with a non-zero `distanceKm`.

The dataset covers 237 countries and territories and is intended for country-level pricing decisions, not mapping. Tiny microstates can resolve to a surrounding country, and points in open water resolve to the nearest coast. Enforce a `distanceKm` limit if open-water matches are unacceptable.

`resolveCountryEndpoint` is available for adapters that implement the `RequestLike`/`ResponseLike` interfaces. It writes status and body through the `Status` and `Body` response values. Most applications should call `resolveCountryByCoordinates` directly, as in the Express example above.

### Evidence levels

| Evidence | Source | Suitable for |
| --- | --- | --- |
| `detected` | Browser geolocation or configured fallback country | Displaying prices |
| `verified` | Billing country, payment provider country, ID check | Checkout |

Browser geolocation is not proof of residency. The client creates `detected` sessions and cannot create `verified` sessions. Only your backend should set `verified`, after an independent check.

The SDK does not verify that a posted country code is truthful. A malicious client can create a session for a cheaper country. Use `requireEvidence: "verified"` for checkout and establish the verified country from a trusted source.

## Security checklist

- [ ] Pricing configuration and payment secrets stay on the server.
- [ ] The client never supplies amount or currency for checkout.
- [ ] `/session` ignores any client-supplied evidence and creates `detected` sessions.
- [ ] Checkout uses `requireEvidence: "verified"` for price-sensitive products.
- [ ] Order amount and currency are persisted and checked again during verification.
- [ ] Payment webhooks verify provider signatures before granting access.
- [ ] Session cookies use `HttpOnly`, `Secure` in production, and an appropriate `SameSite`.
- [ ] Cross-origin CORS uses an exact origin with credentials enabled.
- [ ] Session creation is rate-limited and sessions expire.
- [ ] Raw coordinates and IP addresses are not retained longer than necessary.

## Troubleshooting

**The client returns no prices.**
Check that `/prices/public` exists, returns JSON, and includes the product ids you request. Confirm the resolved country has an entry in your pricing configuration. If the fallback is `require-verification` or `browse-only`, no price is expected until your app verifies the user.

**The session cookie is not sent.**
`sessionCookie` defaults to `Secure`. For local HTTP, pass `{ secure: false }`. For cross-site deployments, use `sameSite: "none"` with `secure: true`. Confirm CORS allows credentials, the API is reachable, and `cookie-parser` (or an equivalent) populates `req.cookies`.

**Checkout returns 403.**
The session may be missing or expired, the product may have no price for the session country, or `requireEvidence: "verified"` is set and the session is still `detected`. Check the session store and upgrade evidence only after verification.

**`verifyPayment` returns `paid: false`.**
The provider's amount or currency does not match the stored order. Check that both sides use minor units and the same currency exponent. Do not compare formatted major-unit strings.

**Prices are 100 times too large or too small.**
The amount is probably in the wrong unit convention. `geo-price` uses ISO 4217 minor units: JPY, KRW, and VND have no decimal places; USD and EUR have two; BHD and KWD have three. Convert at your payment-provider boundary if that provider uses a different convention.

**The detected country is wrong.**
Browser geolocation is not IP geolocation. It reflects the device location and can be affected by GPS accuracy, VPNs, or the user's environment. The local resolver uses simplified borders and can return the nearest coast for points in water. Enforce `distanceKm` limits or offer a verification flow.

**Sessions disappear after a restart.**
`MemorySessionStore` is in-memory only. Use a shared persistent store for deployments with restarts or multiple instances.

## API reference

### `@geo-price/client`

```ts
initGeoPrice(config: GeoPriceClientConfig): Promise<GeoPriceClient>
```

Types:

```ts
type FallbackPolicy =
  | { mode: "default-country"; countryCode: string }
  | { mode: "default-price"; prices: Record<string, RegionalPrice> }
  | { mode: "require-verification" }
  | { mode: "browse-only" };

interface CountryResolver {
  resolve(latitude: number, longitude: number): Promise<string>;
}

interface GeoPriceClientConfig {
  apiUrl: string;
  detection: "browser-geolocation";
  fallback: FallbackPolicy;
  countryResolver?: CountryResolver;
}
```

### `@geo-price/server`

```ts
createPricingSession(input: CreateSessionInput, store: SessionStore): Promise<PricingSession>
sessionCookie(session: PricingSession, options?: Partial<CookieOptions> & { name?: string }): string
clearSessionCookie(options?: Partial<CookieOptions> & { name?: string }): string
createSessionId(): string

calculatePrice(params: CalculatePriceParams): Promise<CalculatedPrice | null>
createPaymentOrder(params: CreatePaymentOrderParams): Promise<PaymentOrder | null>
verifyPayment(params: VerifyPaymentParams): Promise<VerifyPaymentResult>
lookupPrice(config: PricingConfig, productId: ProductId, countryCode: string): Price | null
toMajor(amount: number, currency: string): string

definePricing(config: PricingConfig): PricingConfig
defineRegionalPricing(basePrices, options?): PricingConfig
deriveRegionalPricing(basePrices, options?): PricingConfig
roundToNearest9(amount: number): number

findSessionId(request: RequestLike, cookieName?: string): string | undefined
minorUnitExponent(currency: string): 0 | 2 | 3
roundingUnitExponent(currency: string): 0 | 2 | 3

resolveCountryByCoordinates(latitude: number, longitude: number): ResolvedCountry | null
resolveCountryEndpoint(params: ResolveCountryParams): Promise<void>
currencyForCountry(countryCode: string, fallback: string): string
countryPolygonByCode(code: string): CountryPolygon | undefined
```

Exported data:

```ts
REGIONAL_BASE_COUNTRY
REGIONAL_DEFAULT_RATIO
REGIONAL_RATIOS
COUNTRY_CURRENCIES
COUNTRY_POLYGONS
```

Exported classes and types include `MemorySessionStore`, `SessionStore`, `PricingSession`, `PricingConfig`, `PaymentProvider`, `PaymentOrder`, `CookieOptions`, `RequestLike`, `ResponseLike`, `ResolvedCountry`, and `CountryPolygon`.

## Development

```bash
npm install
npm run build
npm run check
```

`npm run check` runs:

- `check:geo`: resolves 83 representative cities against the bundled polygons.
- `check:pricing`: verifies currency precision and derived pricing across 26 markets.
- `check:api`: verifies sessions, price calculation, checkout orders, and payment verification.

Other repository commands:

```bash
npm run check:geo
npm run check:pricing
npm run check:api
npm pack --dry-run --workspace @geo-price/server
npm pack --dry-run --workspace @geo-price/client
```

The package `prepack` scripts build `dist` automatically. Both packages include only `dist` plus the package metadata and README in the published tarball.

## License

ISC
