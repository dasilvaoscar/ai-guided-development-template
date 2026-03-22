# Backend Architecture — Clean Architecture

This project's backend follows the principles of **Clean Architecture**, proposed by Robert C. Martin. The goal is to keep business rules independent of frameworks, databases, and external services — making the core logic testable in isolation and resilient to infrastructure changes.

## Core Principle: The Dependency Rule

Dependencies always point **inward**. Outer layers know about inner layers; inner layers know nothing about outer layers.

```
┌──────────────────────────────────────────┐
│              Presentation                │  ← HTTP, CLI, WebSocket
│  ┌────────────────────────────────────┐  │
│  │           Infrastructure           │  │  ← DB, external APIs, config
│  │  ┌──────────────────────────────┐  │  │
│  │  │          Application         │  │  │  ← Use cases, DTOs
│  │  │  ┌────────────────────────┐  │  │  │
│  │  │  │         Domain         │  │  │  │  ← Entities, rules, interfaces
│  │  │  └────────────────────────┘  │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

No inner layer may import from an outer layer. Violations of this rule are bugs in the architecture.

---

## Layers

### Domain

The innermost layer. Contains pure business logic and has **zero dependencies** on any framework, library, or infrastructure concern.

**Responsibilities:**
- Define business entities with their identity and behavior
- Define value objects (immutable, equality by value)
- Declare repository interfaces (ports) that outer layers must implement
- Encode domain rules as methods on entities or as domain services
- Define domain-specific error types

**Structure:**
```
src/domain/
  entities/          # Business objects with identity
  value-objects/     # Immutable descriptors
  repositories/      # Interfaces for data access (ports)
  services/          # Logic that spans multiple entities
  errors/            # Domain error classes
```

**Example — repository interface (port):**
```ts
// domain/repositories/user.repository.ts
export interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}
```

The domain defines *what* data access looks like; it never implements it.

---

### Application

Orchestrates domain objects to fulfill discrete use cases. No framework code lives here.

**Responsibilities:**
- Implement one use case per class, following the naming convention `VerbNounUseCase`
- Receive all dependencies (domain ports) via constructor injection
- Define DTOs for use-case input and output
- Handle application-level errors (not domain errors)
- Coordinate cross-cutting concerns like transactions at the use-case boundary

**Structure:**
```
src/application/
  use-cases/         # One class per use case
  dtos/              # Input/output data shapes
  services/          # Cross-cutting application services (e.g. token signing)
```

**Example — use case:**
```ts
// application/use-cases/create-user.use-case.ts
export class CreateUserUseCase {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly hashService: HashService,
  ) {}

  async execute(input: CreateUserDto): Promise<UserOutputDto> {
    const existing = await this.userRepo.findByEmail(input.email);
    if (existing) throw new EmailAlreadyTakenError(input.email);

    const user = User.create({ ...input, password: await this.hashService.hash(input.password) });
    await this.userRepo.save(user);

    return UserOutputDto.fromEntity(user);
  }
}
```

Use cases are the **only** entry point to the domain from the outside.

---

### Infrastructure

Implements the interfaces defined in the domain. Allowed to import any external library or framework.

**Responsibilities:**
- Implement domain repository interfaces using the chosen ORM or query builder
- Set up database connection, ORM models, and migrations
- Implement external service integrations (email, storage, third-party APIs)
- Load and validate environment variables
- Configure dependency injection container

**Structure:**
```
src/infrastructure/
  repositories/      # Concrete implementations of domain repository interfaces
  database/          # ORM models, migrations, connection setup
  http/              # External HTTP clients and third-party integrations
  config/            # Environment variable loading and validation
```

**Example — repository implementation:**
```ts
// infrastructure/repositories/prisma-user.repository.ts
export class PrismaUserRepository implements UserRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string) {
    const row = await this.prisma.user.findUnique({ where: { id } });
    return row ? User.reconstitute(row) : null;
  }

  async save(user: User) {
    await this.prisma.user.upsert({
      where: { id: user.id },
      create: user.toRecord(),
      update: user.toRecord(),
    });
  }
}
```

---

### Presentation

Thin adapter between the delivery mechanism (HTTP, CLI, queue) and the application layer. Contains no business logic.

**Responsibilities:**
- Parse and validate the incoming request
- Call the appropriate use case with a DTO
- Map the use-case output to the response format
- Handle HTTP status codes, headers, and error serialization

**Structure:**
```
src/presentation/
  controllers/       # One controller per resource/domain area
  routes/            # Route definitions wiring URLs to controllers
  middlewares/       # Auth, request validation, error handling
  mappers/           # Transform DTOs to HTTP response shapes
```

**Example — controller:**
```ts
// presentation/controllers/user.controller.ts
export class UserController {
  constructor(private readonly createUser: CreateUserUseCase) {}

  async create(req: Request, res: Response) {
    const output = await this.createUser.execute(req.body);
    res.status(201).json(output);
  }
}
```

If a controller contains conditional logic beyond request parsing, that logic belongs in a use case.

---

## Dependency Injection

All wiring happens at the **composition root** — a single entry-point file that instantiates infrastructure implementations and injects them into use cases and controllers.

```ts
// src/main.ts
const prisma = new PrismaClient();

// Infrastructure
const userRepo = new PrismaUserRepository(prisma);
const hashService = new BcryptHashService();

// Application
const createUser = new CreateUserUseCase(userRepo, hashService);

// Presentation
const userController = new UserController(createUser);

// Framework wiring
app.post("/users", (req, res) => userController.create(req, res));
```

**Rules:**
- No service locators or global singletons.
- Always use constructor injection.
- The composition root is the only place where concrete classes from infrastructure are instantiated.

---

## Dependency Flow Summary

| Layer | May import from | Must not import from |
|---|---|---|
| Domain | Nothing | Application, Infrastructure, Presentation |
| Application | Domain | Infrastructure, Presentation |
| Infrastructure | Domain, Application | Presentation |
| Presentation | Application | Domain (directly), Infrastructure |

---

## Testing Strategy

| Layer | Test type | How |
|---|---|---|
| Domain | Unit | Pure functions, no mocks needed |
| Application | Unit | Mock domain ports with in-memory fakes |
| Infrastructure | Integration | Hit a real database (test container or local) |
| Presentation | Integration/E2E | HTTP client against a running server |
