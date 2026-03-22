# Backend Unit Testing — TypeScript + Vitest

## Stack

- **Runner**: Vitest
- **Assertions**: Vitest built-in (`expect`)
- **Fakes**: plain objects/classes implementing domain ports — no mock libraries needed for unit tests

## Installation

```bash
yarn add -D vitest
# optional — only if coverage is needed:
yarn add -D @vitest/coverage-v8
```

## Configuration

Create `vitest.config.ts` at the project root:

```ts
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    globals: true,
    environment: "node",
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "src") },
  },
});
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

## Test Location

Co-locate tests with the source file they cover, inside a `__tests__/` folder per layer.

```
src/
  domain/
    entities/
      __tests__/
        user.entity.test.ts
  application/
    use-cases/
      __tests__/
        create-user.use-case.test.ts
  infrastructure/
    repositories/
      __tests__/
        prisma-user.repository.test.ts   ← integration, not unit
```

## What and How to Test per Layer

### Domain — pure logic, no fakes needed

Domain entities and value objects have no external dependencies. Test all business rules directly.

```ts
// domain/entities/__tests__/user.entity.test.ts
import { User } from "../user.entity";
import { InvalidEmailError } from "../../errors/invalid-email.error";

it("creates a user with valid data", () => {
  const user = User.create({ name: "Alice", email: "alice@example.com" });
  expect(user.name).toBe("Alice");
  expect(user.email).toBe("alice@example.com");
});

it("throws InvalidEmailError for malformed email", () => {
  expect(() => User.create({ name: "Alice", email: "not-an-email" }))
    .toThrow(InvalidEmailError);
});
```

### Application — use cases with in-memory fakes

Use cases depend on domain ports. Replace them with in-memory fakes — plain classes implementing the same interface.

```ts
// application/use-cases/__tests__/create-user.use-case.test.ts
import { CreateUserUseCase } from "../create-user.use-case";
import { EmailAlreadyTakenError } from "@/domain/errors/email-already-taken.error";
import type { UserRepository } from "@/domain/repositories/user.repository";
import type { HashService } from "@/domain/services/hash.service";

// In-memory fake — implements the port, stores state in memory
class InMemoryUserRepository implements UserRepository {
  private store: Map<string, User> = new Map();

  async findByEmail(email: string) {
    return [...this.store.values()].find(u => u.email === email) ?? null;
  }
  async findById(id: string) {
    return this.store.get(id) ?? null;
  }
  async save(user: User) {
    this.store.set(user.id, user);
  }
  async delete(id: string) {
    this.store.delete(id);
  }
}

class FakeHashService implements HashService {
  async hash(value: string) { return `hashed:${value}`; }
  async compare(value: string, hashed: string) { return hashed === `hashed:${value}`; }
}

let useCase: CreateUserUseCase;
let userRepo: InMemoryUserRepository;

beforeEach(() => {
  userRepo = new InMemoryUserRepository();
  useCase = new CreateUserUseCase(userRepo, new FakeHashService());
});

it("creates and persists a new user", async () => {
  const output = await useCase.execute({ name: "Alice", email: "alice@example.com", password: "secret" });

  expect(output.email).toBe("alice@example.com");
  expect(await userRepo.findByEmail("alice@example.com")).not.toBeNull();
});

it("throws EmailAlreadyTakenError when email is in use", async () => {
  await useCase.execute({ name: "Alice", email: "alice@example.com", password: "secret" });

  await expect(useCase.execute({ name: "Bob", email: "alice@example.com", password: "other" }))
    .rejects.toThrow(EmailAlreadyTakenError);
});
```

### Infrastructure — integration tests (not unit)

Repository implementations require a real database. These are **not** unit tests — run them separately against a test database or container.

```ts
// infrastructure/repositories/__tests__/prisma-user.repository.test.ts
// Tag as integration: run with `vitest run --reporter=verbose` in CI only
```

Keep infrastructure tests out of the default `vitest` watch run by placing them in a separate config or using `testPathPattern` exclusions.

### Presentation — test the controller logic only

Unit-test controllers by calling them directly with mock request/response objects. Do not spin up an HTTP server for unit tests.

```ts
// presentation/controllers/__tests__/user.controller.test.ts
import { UserController } from "../user.controller";
import type { CreateUserUseCase } from "@/application/use-cases/create-user.use-case";

const fakeUseCase = {
  execute: vi.fn().mockResolvedValue({ id: "1", email: "alice@example.com" }),
} satisfies Partial<CreateUserUseCase>;

const mockReq = (body: unknown) => ({ body } as any);
const mockRes = () => {
  const res: any = {};
  res.status = vi.fn().mockReturnValue(res);
  res.json = vi.fn().mockReturnValue(res);
  return res;
};

it("responds 201 with the created user", async () => {
  const controller = new UserController(fakeUseCase as any);
  const res = mockRes();

  await controller.create(mockReq({ name: "Alice", email: "alice@example.com" }), res);

  expect(res.status).toHaveBeenCalledWith(201);
  expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ email: "alice@example.com" }));
});
```

## Rules

- Use in-memory fakes (classes implementing domain ports) over `vi.mock` for dependency inversion — fakes are type-safe and reusable across tests.
- Reserve `vi.fn()` spies for verifying call behavior on collaborators within the same layer.
- Domain tests must never instantiate infrastructure classes.
- Each test must be independent — reset fakes in `beforeEach`, never share mutable state across tests.
- Infrastructure/integration tests must not run in the default unit test suite.
