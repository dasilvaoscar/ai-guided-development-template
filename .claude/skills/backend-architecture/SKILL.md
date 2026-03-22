---
name: backend-architecture
description: Architecture guide for backend and API projects. Use when creating or structuring a backend service, REST API, or server-side application. Defines folder structure, layer responsibilities, and dependency rules based on Clean Architecture.
---

# Backend Architecture — Clean Architecture

## Layers and Responsibilities

Dependencies always point **inward**: `Infrastructure` → `Application` → `Domain`. No layer may import from a layer outside it.

```
src/
  domain/           # Core business rules — no external dependencies
  application/      # Use cases — depends only on domain
  infrastructure/   # Frameworks, DB, external services — implements domain interfaces
  presentation/     # HTTP layer — translates HTTP ↔ application
```

### Domain
The innermost layer. Contains pure business logic with zero framework or library dependencies.

- `entities/` — Business objects with identity and behavior
- `value-objects/` — Immutable objects defined by their attributes
- `repositories/` — **Interfaces** (ports) for data access, defined here but implemented in infrastructure
- `services/` — Domain services for logic that doesn't belong to a single entity
- `errors/` — Domain-specific error classes

```ts
// domain/repositories/user.repository.ts
export interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}
```

### Application
Orchestrates domain objects to fulfill use cases. No framework code.

- `use-cases/` — One class per use case, named `VerbNounUseCase` (e.g. `CreateUserUseCase`)
- `dtos/` — Input/output data shapes for use cases
- `services/` — Application services coordinating multiple use cases or cross-cutting concerns

Each use case receives its dependencies via constructor injection (ports from domain):

```ts
// application/use-cases/create-user.use-case.ts
export class CreateUserUseCase {
  constructor(private readonly userRepo: UserRepository) {}

  async execute(input: CreateUserDto): Promise<UserOutputDto> { ... }
}
```

### Infrastructure
Implements the interfaces (ports) defined in the domain. Allowed to import any external library.

- `repositories/` — Concrete implementations of domain repository interfaces
- `database/` — ORM models, migrations, connection setup
- `http/` — External HTTP clients and third-party API integrations
- `config/` — Environment variable loading and validation

```ts
// infrastructure/repositories/prisma-user.repository.ts
export class PrismaUserRepository implements UserRepository {
  async findById(id: string) { ... }
  async save(user: User) { ... }
}
```

### Presentation
Thin adapter between HTTP and the application layer. No business logic here.

- `controllers/` — Parse request, call use case, map result to response
- `routes/` — Route definitions wiring URLs to controllers
- `middlewares/` — Auth, validation, error handling
- `mappers/` — Transform DTOs to HTTP response shapes

```ts
// presentation/controllers/user.controller.ts
async create(req: Request, res: Response) {
  const output = await this.createUserUseCase.execute(req.body);
  res.status(201).json(output);
}
```

## Dependency Injection

Wire everything at the composition root — a single entry point file that instantiates infrastructure implementations and injects them into use cases:

```ts
// src/main.ts (composition root)
const userRepo = new PrismaUserRepository(prisma);
const createUser = new CreateUserUseCase(userRepo);
const userController = new UserController(createUser);
```

Avoid service locators or global singletons. Prefer constructor injection.

## Rules

- Domain must never import from `application`, `infrastructure`, or `presentation`.
- Application must never import from `infrastructure` or `presentation`.
- Use cases are the **only** entry point to the domain from the outside.
- Controllers must be thin — if a controller has conditionals beyond request parsing, the logic belongs in a use case.
- Repository interfaces live in `domain/repositories/`, their implementations in `infrastructure/repositories/`.
