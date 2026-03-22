---
name: frontend-architecture
description: Architecture guide for frontend projects using Next.js. Use when creating or structuring a Next.js application. Defines a module-based folder structure with dependency inversion for drivers and adapters (HTTP clients, storage, analytics, etc.).
---

# Frontend Architecture — Next.js Modular + Dependency Inversion

## Guiding Principles

- **Module cohesion**: each feature owns its components, hooks, logic, and contracts.
- **Dependency inversion**: modules depend on abstractions (ports), not on concrete drivers (fetch, localStorage, Stripe SDK, etc.).
- **Next.js as a delivery mechanism**: `app/` is only for routing and server concerns — no business logic lives there.

## Folder Structure

```
src/
  app/                    # Next.js App Router — routing only
  modules/                # Feature modules (see below)
  shared/                 # Cross-module UI components, hooks, utils
  drivers/                # Concrete driver implementations (infrastructure)
  lib/                    # App-wide composition root and providers
```

## `app/` — Routing Layer

Only Next.js-specific files live here: `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `route.ts`.

Pages are thin — they import and render the module's entry component:

```ts
// app/checkout/page.tsx
import { CheckoutPage } from "@/modules/checkout/CheckoutPage";
export default CheckoutPage;
```

No business logic, no data fetching logic, no hooks beyond what Next.js expects (`generateMetadata`, `generateStaticParams`).

## `modules/` — Feature Modules

Each module is self-contained. Internal structure is consistent across all modules:

```
modules/
  checkout/
    components/       # UI components used only by this module
    hooks/            # React hooks encapsulating module state/logic
    services/         # Pure functions or classes for business logic
    ports/            # Interfaces (dependency inversion contracts)
    adapters/         # Implementations of ports used within this module
    CheckoutPage.tsx  # Module entry component
    index.ts          # Public API — only export what other modules need
```

Modules must not import from each other's internals. Cross-module communication goes through `index.ts` public exports only.

## Ports and Adapters (Dependency Inversion)

Ports are TypeScript interfaces that define **what** a dependency does, not **how**. Adapters are concrete implementations.

### Defining a Port

```ts
// modules/checkout/ports/payment.port.ts
export interface PaymentPort {
  charge(amount: number, token: string): Promise<{ id: string }>;
}
```

### Implementing an Adapter

```ts
// modules/checkout/adapters/stripe-payment.adapter.ts
import Stripe from "stripe";

export class StripePaymentAdapter implements PaymentPort {
  async charge(amount: number, token: string) {
    const stripe = new Stripe(process.env.STRIPE_KEY!);
    const charge = await stripe.charges.create({ amount, source: token, currency: "usd" });
    return { id: charge.id };
  }
}
```

### Injecting via Hook

```ts
// modules/checkout/hooks/useCheckout.ts
export function useCheckout(payment: PaymentPort) {
  const submit = async (token: string) => {
    await payment.charge(totalAmount, token);
  };
  return { submit };
}
```

## `drivers/` — Global Infrastructure Drivers

Drivers that are shared across modules (not specific to one feature) live here:

```
drivers/
  http/
    fetch.driver.ts         # Implements HttpPort using native fetch
    axios.driver.ts         # Alternative implementation
  storage/
    localStorage.driver.ts  # Implements StoragePort using localStorage
    cookieStorage.driver.ts
  analytics/
    gtm.driver.ts           # Implements AnalyticsPort using GTM
```

Each driver implements a port defined in `shared/ports/`:

```ts
// shared/ports/http.port.ts
export interface HttpPort {
  get<T>(url: string): Promise<T>;
  post<T>(url: string, body: unknown): Promise<T>;
}
```

```ts
// drivers/http/fetch.driver.ts
export class FetchHttpDriver implements HttpPort {
  async get<T>(url: string): Promise<T> {
    const res = await fetch(url);
    return res.json();
  }
  async post<T>(url: string, body: unknown): Promise<T> {
    const res = await fetch(url, { method: "POST", body: JSON.stringify(body) });
    return res.json();
  }
}
```

## `lib/` — Composition Root

Wire ports to their concrete drivers here, not inside components or hooks:

```ts
// lib/container.ts
import { FetchHttpDriver } from "@/drivers/http/fetch.driver";
import { StripePaymentAdapter } from "@/modules/checkout/adapters/stripe-payment.adapter";

export const httpDriver = new FetchHttpDriver();
export const paymentAdapter = new StripePaymentAdapter();
```

Provide them via React Context or pass as props at the page level:

```tsx
// lib/providers.tsx
export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <PaymentContext.Provider value={paymentAdapter}>
      <HttpContext.Provider value={httpDriver}>
        {children}
      </HttpContext.Provider>
    </PaymentContext.Provider>
  );
}
```

## Rules

- `app/` files must not contain business logic or import from `drivers/` directly.
- Modules must not instantiate drivers themselves — receive them via props, context, or hooks.
- Ports (interfaces) live close to where they are consumed: module-specific ports in `modules/<name>/ports/`, shared ports in `shared/ports/`.
- Adapters and drivers are swappable — tests replace them with in-memory fakes implementing the same port.
- `index.ts` is the module boundary — never import `modules/checkout/hooks/useCheckout` from another module directly.
