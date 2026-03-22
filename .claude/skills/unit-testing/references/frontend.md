# Frontend Unit Testing — Vitest + React Testing Library

## Stack

- **Runner**: Vitest
- **Component testing**: React Testing Library (`@testing-library/react`)
- **Assertions**: Vitest built-in (`expect`) + `@testing-library/jest-dom`
- **User interaction**: `@testing-library/user-event`

## Installation

```bash
yarn add -D vitest @vitejs/plugin-react jsdom \
  @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

## Configuration

Create `vitest.config.ts` at the project root:

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./tests/setup.ts"],
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "src") },
  },
});
```

Create `tests/setup.ts`:

```ts
import "@testing-library/jest-dom";
```

Add to `package.json`:

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage"
  }
}
```

## What to Test

Follow the module structure. Each module has its own `__tests__/` folder next to what it tests.

```
modules/
  checkout/
    hooks/
      __tests__/
        useCheckout.test.ts
    services/
      __tests__/
        calculateTotal.test.ts
    components/
      __tests__/
        CheckoutForm.test.tsx
```

### Hooks — inject port fakes via parameter

Because hooks receive ports as parameters (dependency inversion), tests replace them with in-memory fakes. No mocking libraries needed.

```ts
// modules/checkout/hooks/__tests__/useCheckout.test.ts
import { renderHook, act } from "@testing-library/react";
import { useCheckout } from "../useCheckout";
import type { PaymentPort } from "../../ports/payment.port";

const fakePayment: PaymentPort = {
  charge: vi.fn().mockResolvedValue({ chargeId: "fake-123" }),
  refund: vi.fn().mockResolvedValue(undefined),
};

it("calls payment.charge with the correct amount", async () => {
  const { result } = renderHook(() => useCheckout(fakePayment));

  await act(async () => {
    await result.current.submit("tok_test");
  });

  expect(fakePayment.charge).toHaveBeenCalledWith({
    amount: expect.any(Number),
    currency: "usd",
    token: "tok_test",
  });
});
```

### Services — pure function tests

```ts
// modules/checkout/services/__tests__/calculateTotal.test.ts
import { calculateTotal } from "../calculateTotal";

it("sums item prices and applies discount", () => {
  const items = [{ price: 100 }, { price: 50 }];
  expect(calculateTotal(items, 0.1)).toBe(135);
});

it("returns 0 for an empty cart", () => {
  expect(calculateTotal([], 0)).toBe(0);
});
```

### Components — behavior, not implementation

Test what the user sees and does. Do not test internal state or implementation details.

```tsx
// modules/checkout/components/__tests__/CheckoutForm.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { CheckoutForm } from "../CheckoutForm";

const mockSubmit = vi.fn();

beforeEach(() => mockSubmit.mockClear());

it("calls onSubmit when form is submitted with valid data", async () => {
  render(<CheckoutForm onSubmit={mockSubmit} />);

  await userEvent.type(screen.getByLabelText(/card number/i), "4242424242424242");
  await userEvent.click(screen.getByRole("button", { name: /pay/i }));

  expect(mockSubmit).toHaveBeenCalledTimes(1);
});

it("shows validation error when card number is empty", async () => {
  render(<CheckoutForm onSubmit={mockSubmit} />);

  await userEvent.click(screen.getByRole("button", { name: /pay/i }));

  expect(screen.getByText(/card number is required/i)).toBeInTheDocument();
  expect(mockSubmit).not.toHaveBeenCalled();
});
```

### Drivers — test the transformation, mock the I/O

```ts
// drivers/http/__tests__/fetch.driver.test.ts
import { FetchHttpDriver } from "../fetch.driver";

it("throws HttpError when response is not ok", async () => {
  global.fetch = vi.fn().mockResolvedValue({ ok: false, status: 404, text: async () => "Not Found" });

  const driver = new FetchHttpDriver("https://api.example.com");
  await expect(driver.get("/missing")).rejects.toThrow("404");
});
```

## Rules

- Use fakes (objects implementing the port interface) over mocks for ports and adapters.
- Never test implementation details — test observable behavior.
- One `it` block per behavior; keep tests short and independent.
- Avoid `beforeAll` with shared mutable state — prefer `beforeEach`.
- Do not test Next.js routing or server components with unit tests — use E2E for those.
