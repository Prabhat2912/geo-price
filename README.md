# geo-price

**Regional pricing for the global web.**

`geo-price` is a TypeScript SDK for displaying country-specific prices
and creating server-controlled checkout orders. Configure prices per
market instead of deriving them from live exchange rates.

- **Client:** `@geo-price/client` --- browser location permission,
  session setup, and price display.
- **Server:** `@geo-price/server` --- country resolution, pricing
  sessions, configured prices, and payment-order helpers.
- **Framework-independent:** no mandatory React, Express, or
  payment-provider dependency.

> **Security note:** Browser geolocation is a location signal, not proof
> of residency. Treat client-detected country information as unverified.
> For price-sensitive checkout, verify eligibility independently on your
> server.

## Install

```bash
npm install @geo-price/client @geo-price/server
```

Both packages are ESM-only and include TypeScript declarations. The
server package requires Node.js 18 or newer. Browser geolocation
requires a supported browser and a secure context (HTTPS in production;
localhost is generally permitted for development).

## How it works

```text
Browser                 Your backend                 Payment provider
   |                          |                              |
   |-- location permission    |                              |
   |-- coordinates ---------->| resolve country             |
   |<--------------------------| country code                 |
   |-- create session ------->| set HttpOnly cookie          |
   |-- request display prices>| use server pricing config    |
   |<--------------------------| public price list            |
   |-- checkout request ----->| calculate amount server-side >|
   |                          |<--------- payment result -----|
```

The SDK does not silently switch to IP-based geolocation when permission
is denied. Your application must configure what happens when location
detection fails.

## Quick start

### 1. Configure prices on the server

```ts
import { definePricing } from "@geo-price/server";

export const pricing = definePricing({
  premium: {
    US: { amount: 599, currency: "USD" }, // $5.99
    GB: { amount: 479, currency: "GBP" }, // £4.79
    IN: { amount: 17979, currency: "INR" }, // ₹179.79
    JP: { amount: 899, currency: "JPY" }, // ¥899
  },
});
```

Amounts use ISO 4217 minor units: for example, `599` represents USD
5.99, while `899` represents JPY 899. Configure market prices
explicitly; these amounts are examples, not recommended price points.

### 2. Initialize the browser client

```ts
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

`apiUrl` must point to your backend implementation of the SDK's client
endpoint contract. The client needs these routes:

---

Endpoint Purpose

---

`POST /geo/resolve` Accept coordinates and return
`{ "countryCode": "IN" }` (or the
resolved country code).

`POST /session` Create a server-managed session and
set the session cookie.

`GET /prices/public?countryCode=IN` Return configured display prices
for the requested country.

---

These are routes your application implements; installing the packages
does not automatically create HTTP endpoints. See the integration
documentation in the repository for complete route examples.

### 3. Display a price

```ts
const price = await geoPrice.getPrice("premium");

if (price) {
  console.log(`${price.major} ${price.currencyCode}`);
}
```

The client reads the price cached during initialization. If no price is
available, handle that state in your UI.

### 4. Create checkout on the server

Use the server package's `createPaymentOrder` helper with your
server-side pricing configuration, session store, and payment-provider
adapter. **Never accept the amount or currency from the browser as
checkout authority.** Persist the created order and verify the
provider's response against the stored amount and currency before
granting access.

The package provides integration helpers; you still need to implement
your HTTP routes, persistent storage, provider adapter, and
application-specific access logic.

## Security and trust model

- **`detected` evidence** comes from browser geolocation or a
  configured fallback. It is suitable for displaying prices, not
  proving residency.
- **`verified` evidence** must be assigned by your backend after an
  independent check, such as an appropriate billing-country or
  payment-provider verification.
- The client cannot establish verified evidence. Do not accept an
  `evidence: "verified"` value from a request body.
- Checkout should use server-side session and pricing data. Store
  order details and compare the provider-reported amount and currency
  during payment verification.
- Use an `HttpOnly` session cookie, `Secure` in production, correct
  `SameSite` settings, and exact-origin CORS when credentials are
  enabled.
- Avoid retaining raw coordinates longer than necessary.

The SDK does not itself prove a user's residence or prevent a determined
user from submitting another country code. If regional price eligibility
is sensitive, implement independent verification and enforce it on the
server.

## Fallback behavior

A fallback policy is required so the application explicitly decides what
happens when location detection or resolution fails.

---

Mode Behavior

---

`default-country` Use a configured country and create
a detected session.

`default-price` Return configured local fallback
prices without creating a session.

`require-verification` Return no price until your
application establishes a session
through its verification flow.

`browse-only` Return no price; your application
decides whether checkout is
unavailable.

---

`require-verification` and `browse-only` behave the same inside the SDK;
the distinction is how your application handles access and checkout.

## Country resolution

`@geo-price/server` includes a local coordinate-to-country resolver
backed by Natural Earth country polygons. It requires no API key and
makes no network calls. Because the dataset uses simplified borders,
tiny territories and points near coastlines may not resolve precisely.
Apply a distance limit or a separate verification flow where
appropriate.

## Regional pricing

For production, prefer explicit per-country prices that you have
reviewed for your business. The server package also provides
`defineRegionalPricing` for deriving initial prices from a base price
and local-price multipliers.

Those multipliers are **not exchange rates** and are illustrative
starting points, not validated purchasing-power recommendations. Review
the resulting prices before using them with customers.

## Development

```bash
npm install
npm run build
npm run check
```

`npm run check` runs the repository's country-resolution, pricing, and
API checks.

## Documentation

Keep detailed material in focused documents rather than duplicating it
here:

- Integration guide: complete Express routes, cookies, CORS, and
  production session stores.
- Client API: configuration, fallback types, methods, and errors.
- Server API: sessions, pricing, payment-provider interface, and
  exported utilities.
- Pricing guide: currency precision, regional ratios, overrides, and
  rounding.
- Security guide: evidence levels, verification, checkout, and
  deployment checklist.
- Troubleshooting: common integration and runtime failures.

## License

ISC
