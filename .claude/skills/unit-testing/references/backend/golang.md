# Backend Unit Testing — Go

## Stack

- **Runner**: Go built-in `testing` package
- **Assertions**: `github.com/stretchr/testify` (`assert` and `require`)
- **Fakes**: hand-written structs implementing domain interfaces — no mock generation required for unit tests

## Installation

```bash
go get github.com/stretchr/testify
go mod tidy
```

Add to `Makefile`:

```makefile
test:
	go test ./... -v -race

test/unit:
	go test ./internal/... ./cmd/... -v -race

test/cover:
	go test ./... -coverprofile=coverage.out && go tool cover -html=coverage.out
```

## Test Location

Place test files alongside the source file they cover, using the `_test.go` suffix. Use the `_test` package suffix to test only the exported API (black-box), or the same package name to access unexported symbols when needed.

```
internal/
  domain/
    user.go
    user_test.go
  application/
    create_user.go
    create_user_test.go
  infrastructure/
    postgres_user_repo.go
    postgres_user_repo_test.go   ← integration, not unit
```

## What and How to Test per Layer

### Domain — pure logic, no dependencies

Domain types and functions have no external dependencies. Test all business rules directly.

```go
// internal/domain/user_test.go
package domain_test

import (
	"testing"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"yourmodule/internal/domain"
)

func TestNewUser_ValidData(t *testing.T) {
	user, err := domain.NewUser("Alice", "alice@example.com")
	require.NoError(t, err)
	assert.Equal(t, "Alice", user.Name)
	assert.Equal(t, "alice@example.com", user.Email)
}

func TestNewUser_InvalidEmail(t *testing.T) {
	_, err := domain.NewUser("Alice", "not-an-email")
	assert.ErrorIs(t, err, domain.ErrInvalidEmail)
}
```

Use `require` when the test cannot continue after a failure (e.g. nil pointer risk). Use `assert` to collect all failures.

### Application — use cases with in-memory fakes

Use cases depend on domain interfaces (ports). Replace them with in-memory structs that implement the same interface.

```go
// internal/application/create_user_test.go
package application_test

import (
	"context"
	"testing"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"yourmodule/internal/application"
	"yourmodule/internal/domain"
)

// InMemoryUserRepo implements domain.UserRepository in memory
type InMemoryUserRepo struct {
	store map[string]domain.User
}

func newInMemoryUserRepo() *InMemoryUserRepo {
	return &InMemoryUserRepo{store: make(map[string]domain.User)}
}

func (r *InMemoryUserRepo) FindByEmail(_ context.Context, email string) (*domain.User, error) {
	for _, u := range r.store {
		if u.Email == email {
			return &u, nil
		}
	}
	return nil, nil
}

func (r *InMemoryUserRepo) Save(_ context.Context, u domain.User) error {
	r.store[u.ID] = u
	return nil
}

// FakeHasher implements domain.Hasher
type FakeHasher struct{}

func (h *FakeHasher) Hash(value string) (string, error)             { return "hashed:" + value, nil }
func (h *FakeHasher) Compare(value, hashed string) (bool, error)    { return hashed == "hashed:"+value, nil }

func TestCreateUser_Succeeds(t *testing.T) {
	repo := newInMemoryUserRepo()
	uc := application.NewCreateUserUseCase(repo, &FakeHasher{})

	output, err := uc.Execute(context.Background(), application.CreateUserInput{
		Name:     "Alice",
		Email:    "alice@example.com",
		Password: "secret",
	})

	require.NoError(t, err)
	assert.Equal(t, "alice@example.com", output.Email)

	saved, _ := repo.FindByEmail(context.Background(), "alice@example.com")
	require.NotNil(t, saved)
}

func TestCreateUser_DuplicateEmail(t *testing.T) {
	repo := newInMemoryUserRepo()
	uc := application.NewCreateUserUseCase(repo, &FakeHasher{})

	input := application.CreateUserInput{Name: "Alice", Email: "alice@example.com", Password: "secret"}
	_, err := uc.Execute(context.Background(), input)
	require.NoError(t, err)

	_, err = uc.Execute(context.Background(), input)
	assert.ErrorIs(t, err, domain.ErrEmailAlreadyTaken)
}
```

### Infrastructure — integration tests (not unit)

Repository implementations require a real database. Tag them with a build tag or test name convention so they are excluded from the default unit run.

```go
//go:build integration

package infrastructure_test
// Run with: go test -tags=integration ./internal/infrastructure/...
```

Or use a skip guard:

```go
func TestPostgresUserRepo_Save(t *testing.T) {
	if testing.Short() {
		t.Skip("skipping integration test in short mode")
	}
	// ...
}
```

Run unit tests only: `go test -short ./...`

### Presentation (HTTP handlers) — httptest

Test HTTP handlers in isolation using `net/http/httptest`. No real server needed.

```go
// internal/presentation/user_handler_test.go
package presentation_test

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"github.com/stretchr/testify/assert"
	"yourmodule/internal/presentation"
)

type fakeCreateUserUseCase struct{}

func (f *fakeCreateUserUseCase) Execute(_ context.Context, input application.CreateUserInput) (application.CreateUserOutput, error) {
	return application.CreateUserOutput{ID: "1", Email: input.Email}, nil
}

func TestCreateUserHandler_Returns201(t *testing.T) {
	handler := presentation.NewUserHandler(&fakeCreateUserUseCase{})

	body := `{"name":"Alice","email":"alice@example.com","password":"secret"}`
	req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()

	handler.Create(rec, req)

	assert.Equal(t, http.StatusCreated, rec.Code)
	assert.Contains(t, rec.Body.String(), "alice@example.com")
}
```

## Table-Driven Tests

Prefer table-driven tests for logic with multiple input/output cases:

```go
func TestCalculateDiscount(t *testing.T) {
	tests := []struct {
		name     string
		price    float64
		discount float64
		want     float64
	}{
		{"10% off", 100, 0.10, 90},
		{"no discount", 50, 0, 50},
		{"full discount", 200, 1.0, 0},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			got := domain.ApplyDiscount(tc.price, tc.discount)
			assert.Equal(t, tc.want, got)
		})
	}
}
```

## Rules

- Use hand-written in-memory fakes over mock generators (e.g. `mockery`) for unit tests — fakes are explicit and type-checked by the compiler.
- Use `require` for assertions where a nil/zero value would cause a panic in subsequent lines; use `assert` otherwise.
- Integration tests must be gated behind `-tags=integration` or `-short` so they never run in the unit test suite.
- Each test must be independent — initialize all fakes inside the test or `TestMain`; never share mutable state across tests.
- Use `t.Parallel()` in tests that have no shared mutable state to speed up the suite.
