# Frontend Architecture — Next.js Modular + Ports & Adapters

This project's frontend follows a **module-based architecture** on top of Next.js App Router, combined with the **Ports & Adapters** pattern (also known as Hexagonal Architecture) for all external dependencies (HTTP clients, storage, analytics, payment gateways, etc.).

## Core Principles

- **Module cohesion**: each feature owns its components, hooks, logic, and contracts.
- **Dependency inversion**: modules depend on abstractions (ports), never on concrete drivers.
- **Next.js as a delivery mechanism**: `app/` handles routing and server concerns only — no business logic lives there.
- **Explicit boundaries**: modules communicate only through their public `index.ts` exports.

---

## Folder Structure

```
src/
  app/          # Next.js App Router — routing only
  modules/      # Feature modules
  shared/       # Cross-module UI components, hooks, utilities, and shared ports
  drivers/      # Concrete driver implementations (infrastructure)
  lib/          # Composition root, React providers, app-wide setup
```

---

## Layers

### `app/` — Routing Layer

Only Next.js-specific files live here: `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `route.ts` (API routes), and Next.js metadata exports.

Pages are thin shells that import and render the module's entry component:

```tsx
// app/checkout/page.tsx
import { CheckoutPage } from "@/modules/checkout/CheckoutPage";
export default CheckoutPage;
```

**Rules:**
- No business logic.
- No direct imports from `drivers/`.
- No React hooks beyond what Next.js expects (`generateMetadata`, `generateStaticParams`).
- Server Components may fetch data using injected drivers from `lib/`, but must not instantiate drivers themselves.

---

### `modules/` — Feature Modules

Each module is a self-contained vertical slice of the application. Every module follows the same internal structure:

```
modules/
  checkout/
    components/       # UI components private to this module
    hooks/            # React hooks encapsulating module state and logic
    services/         # Pure functions or classes for business logic
    ports/            # TypeScript interfaces (contracts) for external dependencies
    adapters/         # Implementations of ports specific to this module
    CheckoutPage.tsx  # Module entry component
    index.ts          # Public API — only export what other modules need
```

**Module boundary rule:** modules must not import from each other's internals. Cross-module access is only allowed through the `index.ts` public export.

```ts
// ✅ Allowed
import { useCartSummary } from "@/modules/cart";

// ❌ Not allowed — bypasses the module boundary
import { useCartSummary } from "@/modules/cart/hooks/useCartSummary";
```

---

### `shared/` — Cross-Module Concerns

Contains code that is genuinely reusable across multiple modules but does not belong to any single one.

```
shared/
  components/    # Generic UI components (Button, Modal, Input, etc.)
  hooks/         # Generic React hooks (useDebounce, useMediaQuery, etc.)
  utils/         # Pure utility functions
  ports/         # Shared port interfaces used by multiple modules
```

**Rule:** `shared/` must not import from any module. Dependency is one-directional: modules may import from `shared/`, never the reverse.

---

### Ports & Adapters

Ports are TypeScript interfaces that define **what** a dependency does, without specifying **how**. Adapters are concrete implementations.

This pattern means modules are never coupled to a specific library or SDK — only to the contract.

#### Defining a Port

Ports are defined close to where they are consumed:
- Module-specific ports → `modules/<name>/ports/`
- Shared ports (used by multiple modules) → `shared/ports/`

```ts
// modules/checkout/ports/payment.port.ts
export interface PaymentPort {
  charge(params: { amount: number; currency: string; token: string }): Promise<{ chargeId: string }>;
  refund(chargeId: string): Promise<void>;
}
```

```ts
// shared/ports/http.port.ts
export interface HttpPort {
  get<T>(url: string, options?: RequestOptions): Promise<T>;
  post<T>(url: string, body: unknown, options?: RequestOptions): Promise<T>;
  put<T>(url: string, body: unknown, options?: RequestOptions): Promise<T>;
  delete<T>(url: string, options?: RequestOptions): Promise<T>;
}
```

#### Implementing an Adapter

Module-specific adapters live in `modules/<name>/adapters/`. Adapters that wrap a shared driver live in `drivers/`.

```ts
// modules/checkout/adapters/stripe-payment.adapter.ts
import Stripe from "stripe";
import type { PaymentPort } from "../ports/payment.port";

export class StripePaymentAdapter implements PaymentPort {
  private stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

  async charge({ amount, currency, token }) {
    const charge = await this.stripe.charges.create({ amount, currency, source: token });
    return { chargeId: charge.id };
  }

  async refund(chargeId: string) {
    await this.stripe.refunds.create({ charge: chargeId });
  }
}
```

#### Consuming via Hook

Modules receive their adapters via props or React Context — never by instantiating them internally:

```ts
// modules/checkout/hooks/useCheckout.ts
import type { PaymentPort } from "../ports/payment.port";

export function useCheckout(payment: PaymentPort) {
  const submit = async (token: string) => {
    const { chargeId } = await payment.charge({ amount: total, currency: "usd", token });
    // proceed with chargeId...
  };
  return { submit };
}
```

---

### `drivers/` — Global Infrastructure Drivers

Drivers that are shared across modules live here. Each driver implements a port from `shared/ports/`.

```
drivers/
  http/
    fetch.driver.ts          # HttpPort implemented with native fetch
    axios.driver.ts          # HttpPort implemented with Axios
  storage/
    localStorage.driver.ts   # StoragePort using window.localStorage
    cookieStorage.driver.ts  # StoragePort using cookies
  analytics/
    gtm.driver.ts            # AnalyticsPort using Google Tag Manager
    segment.driver.ts        # AnalyticsPort using Segment
```

**Example — FetchHttpDriver:**

```ts
// drivers/http/fetch.driver.ts
import type { HttpPort } from "@/shared/ports/http.port";

export class FetchHttpDriver implements HttpPort {
  constructor(private readonly baseUrl: string) {}

  async get<T>(url: string): Promise<T> {
    const res = await fetch(`${this.baseUrl}${url}`);
    if (!res.ok) throw new HttpError(res.status, await res.text());
    return res.json();
  }

  async post<T>(url: string, body: unknown): Promise<T> {
    const res = await fetch(`${this.baseUrl}${url}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });
    if (!res.ok) throw new HttpError(res.status, await res.text());
    return res.json();
  }
}
```

Swapping `fetch` for `axios` only requires creating a new driver and updating the composition root — no module code changes.

---

### `lib/` — Composition Root and Providers

This is where concrete drivers and adapters are instantiated and provided to the application. No module instantiates its own dependencies.

```ts
// lib/container.ts
import { FetchHttpDriver } from "@/drivers/http/fetch.driver";
import { LocalStorageDriver } from "@/drivers/storage/localStorage.driver";
import { GtmAnalyticsDriver } from "@/drivers/analytics/gtm.driver";
import { StripePaymentAdapter } from "@/modules/checkout/adapters/stripe-payment.adapter";

export const http = new FetchHttpDriver(process.env.NEXT_PUBLIC_API_URL!);
export const storage = new LocalStorageDriver();
export const analytics = new GtmAnalyticsDriver(process.env.NEXT_PUBLIC_GTM_ID!);
export const payment = new StripePaymentAdapter();
```

Drivers and adapters are distributed via React Context:

```tsx
// lib/providers.tsx
import { http, storage, analytics, payment } from "./container";

export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <HttpContext.Provider value={http}>
      <StorageContext.Provider value={storage}>
        <AnalyticsContext.Provider value={analytics}>
          <PaymentContext.Provider value={payment}>
            {children}
          </PaymentContext.Provider>
        </AnalyticsContext.Provider>
      </StorageContext.Provider>
    </HttpContext.Provider>
  );
}
```

---

## Dependency Flow Summary

```
app/  ──→  modules/  ──→  shared/
            │                │
            ↓                ↓
         adapters/        drivers/
            │                │
            └──── lib/ ──────┘
                 (composition root)
```

| Layer | May import from | Must not import from |
|---|---|---|
| `app/` | `modules/` (index), `lib/` | `drivers/` directly, module internals |
| `modules/` | `shared/`, own `ports/`, own `adapters/` | Other module internals, `drivers/` directly |
| `shared/` | Nothing from this project | `modules/`, `drivers/`, `lib/` |
| `drivers/` | `shared/ports/`, external libraries | `modules/`, `app/` |
| `lib/` | Everything (composition root) | — |

---

## Testing Strategy

| Layer | Test type | How |
|---|---|---|
| Module services/hooks | Unit | Inject in-memory fakes that implement the port |
| Adapters | Integration | Test against the real external SDK (sandboxed) |
| Drivers | Unit | Test the transformation logic; mock `fetch`/SDK |
| Pages | E2E | Playwright or Cypress with the full app running |

Because modules depend only on ports (interfaces), replacing a real adapter with a fake in tests requires no changes to the module code:

```ts
// test fake — implements the same port, no SDK needed
class FakePaymentAdapter implements PaymentPort {
  async charge() { return { chargeId: "fake-123" }; }
  async refund() {}
}
```
