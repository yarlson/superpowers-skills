---
name: Go Code Writing
description: Write idiomatic, performant Go code following best practices for structure, patterns, and tooling
when_to_use: when writing Go code, building Go projects, or debugging Go applications - covers idioms, performance, project structure, testing, and ecosystem choices
version: 1.0.0
languages: go
---

# Go Code Writing

## Overview

Write Go code that is simple, explicit, and idiomatic. Go values clarity over cleverness, composition over inheritance, and explicit error handling over exceptions.

**Core Philosophy:**
- Simplicity beats cleverness
- Explicit beats implicit (especially errors)
- Composition beats inheritance
- Clear beats clever

## Quick Reference

| Pattern | Idiomatic | Anti-pattern |
|---------|-----------|--------------|
| **Error handling** | `if err != nil { return fmt.Errorf("context: %w", err) }` | `panic()`, ignoring errors |
| **Nil checks** | `if obj == nil { return }` | Assuming non-nil |
| **Constructors** | `func NewThing() *Thing` | Exported uninitialized structs |
| **Interfaces** | Accept interfaces, return concrete types | Return interfaces |
| **Interface location** | Consumer defines near usage | Producer exports in same package |
| **Goroutines** | For I/O, with bounded concurrency | For CPU work without limits |
| **Imports** | stdlib → external → internal (grouped) | Random ordering, no grouping |
| **Testing** | Use testify for assertions/mocks | stdlib testing package |

## Import Organization

**Always group imports in three sections:**

```go
import (
    // Group 1: Standard library (alphabetical)
    "context"
    "fmt"
    "net/http"

    // Group 2: External dependencies (alphabetical)
    "github.com/gin-gonic/gin"
    "gorm.io/gorm"

    // Group 3: Internal packages (alphabetical)
    "yourproject/internal/config"
    "yourproject/internal/models"
)
```

**Tools:** `goimports` handles this automatically. Always run before committing.

## Error Handling

**Always check errors. Always wrap with context:**

```go
// ✅ Good: Check and wrap
user, err := repo.GetByID(id)
if err != nil {
    return nil, fmt.Errorf("failed to get user %d: %w", id, err)
}

// ❌ Bad: Ignoring
user, _ := repo.GetByID(id)

// ❌ Bad: No context
if err != nil {
    return nil, err
}
```

**Sentinel errors for control flow:**

```go
var ErrNotFound = errors.New("not found")

// In function
if user == nil {
    return nil, ErrNotFound
}

// In caller
user, err := repo.GetByID(id)
if errors.Is(err, ErrNotFound) {
    // Handle not found specifically
}
```

## Interfaces

**Consumer defines interfaces, not producers:**

```go
// ✅ Good: Consumer package defines what it needs
package handlers

type UserGetter interface {
    GetByID(id int) (*User, error)
}

func NewUserHandler(getter UserGetter) *UserHandler {
    return &UserHandler{getter: getter}
}
```

```go
// ❌ Bad: Producer exports interface
package database

type UserRepository interface {
    GetByID(id int) (*User, error)
}

type PostgresUserRepo struct{}
func (r *PostgresUserRepo) GetByID(id int) (*User, error) { ... }
```

**Accept interfaces, return concrete types:**

```go
// ✅ Good
func NewService(repo UserRepository) *Service { ... }
func (s *Service) GetUser(id int) (*User, error) { ... }

// ❌ Bad: Returning interface
func (s *Service) GetUser(id int) (UserInterface, error) { ... }
```

**Keep interfaces small:**

```go
// ✅ Good: Focused interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

// ❌ Bad: Too many methods
type DataStore interface {
    Create(...) error
    Read(...) error
    Update(...) error
    Delete(...) error
    List(...) error
    Count() int
    // ... 10 more methods
}
```

## Struct Patterns

**Use constructors for complex initialization:**

```go
type Server struct {
    addr string
    db   *sql.DB
}

// ✅ Good: Constructor validates and initializes
func NewServer(addr string, db *sql.DB) (*Server, error) {
    if addr == "" {
        return nil, errors.New("addr cannot be empty")
    }
    if db == nil {
        return nil, errors.New("db cannot be nil")
    }
    return &Server{addr: addr, db: db}, nil
}
```

**Functional options for many parameters:**

```go
type ServerOption func(*Server)

func WithTimeout(d time.Duration) ServerOption {
    return func(s *Server) {
        s.timeout = d
    }
}

func NewServer(addr string, opts ...ServerOption) *Server {
    s := &Server{addr: addr, timeout: 30 * time.Second}
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage
srv := NewServer(":8080", WithTimeout(60*time.Second))
```

## Performance Quick Wins

**Pointer vs Value:**
- Use pointers for large structs (>64 bytes) or when mutation needed
- Use values for small structs (primitives, time.Time)
- Methods that modify: pointer receivers
- Methods that read only: value receivers for small types, pointer for large

```go
// ✅ Good: Large struct, pointer receiver
type User struct { /* many fields */ }
func (u *User) UpdateEmail(email string) { u.Email = email }

// ✅ Good: Small struct, value receiver for read-only
type Point struct { X, Y int }
func (p Point) Distance() float64 { return math.Sqrt(float64(p.X*p.X + p.Y*p.Y)) }
```

**Preallocate slices and maps:**

```go
// ✅ Good: Preallocate when size known
users := make([]User, 0, 100)  // cap=100
counts := make(map[string]int, 50)  // hint=50

// ❌ Bad: Reallocates repeatedly
var users []User  // starts at 0, grows gradually
```

**String concatenation:**

```go
// ✅ Good: Use strings.Builder for multiple concatenations
var b strings.Builder
for _, s := range items {
    b.WriteString(s)
}
result := b.String()

// ❌ Bad: Creates many intermediate strings
result := ""
for _, s := range items {
    result += s  // Allocates new string each iteration
}
```

## Technology Stack

### HTTP Servers

**Decision tree:**
- Small projects, simple routing → `net/http` (stdlib)
- Production APIs, complex routing, middleware → `github.com/gin-gonic/gin`

```go
// Small project: stdlib
http.HandleFunc("/health", healthHandler)
http.ListenAndServe(":8080", nil)

// Production: gin
router := gin.Default()
router.GET("/health", healthHandler)
router.Run(":8080")
```

### Database

**Always use GORM:** `gorm.io/gorm`

```go
import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
```

See `@go-ecosystem.md` for GORM patterns and best practices.

### CLI Tools

**Always use Cobra:** `github.com/spf13/cobra`

```go
var rootCmd = &cobra.Command{
    Use:   "myapp",
    Short: "My application",
}

func Execute() error {
    return rootCmd.Execute()
}
```

### Configuration

**For .env files:** `github.com/joho/godotenv`

```go
import "github.com/joho/godotenv"

func init() {
    _ = godotenv.Load() // Load .env file, ignore error if not found
}
```

**For complex config:** `github.com/spf13/viper`
**For struct binding:** `github.com/kelseyhightower/envconfig`

### Testing

**Always use testify:** `github.com/stretchr/testify`

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestUserService(t *testing.T) {
    user, err := service.GetUser(1)
    require.NoError(t, err)  // Stops test if error
    assert.Equal(t, "Alice", user.Name)  // Continues even if fails
}
```

**For mocking, use moq:** `github.com/matryer/moq`

```go
//go:generate moq -out user_repo_mock.go . UserRepository

type UserRepository interface {
    GetByID(id int) (*User, error)
}
```

See `@go-tooling.md` for testing tools (coverage, race detector, benchmarks).

### Logging

**Prefer slog (stdlib):** `log/slog` (Go 1.21+)

```go
import "log/slog"

slog.Info("user created", "id", user.ID, "name", user.Name)
slog.Error("failed to save user", "error", err)
```

**For high performance:** `github.com/rs/zerolog` or `go.uber.org/zap`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Goroutine leaks | Always close channels, use context for cancellation |
| Pointer to loop variable | Copy: `item := item` before goroutine (Go <1.22) |
| Ignoring deferred Close() errors | `defer func() { if err := f.Close(); err != nil { ... } }()` |
| Overusing `init()` | Use explicit constructors instead |
| Using `any` everywhere | Use concrete types or generics |
| Not running `go fmt` | Set up editor to format on save |
| Not checking race conditions | Run `go test -race` regularly |
| Using stdlib `testing` | Use `testify` for better assertions |
| Using `gomock` | Use `moq` instead (simpler, generates real code) |

## Context Propagation

**Always pass context as first parameter:**

```go
// ✅ Good
func (s *Service) GetUser(ctx context.Context, id int) (*User, error) {
    return s.repo.GetByID(ctx, id)
}

// ❌ Bad: No context
func (s *Service) GetUser(id int) (*User, error) { ... }
```

**Use context for:**
- Request-scoped values (trace IDs, user info)
- Cancellation signals
- Timeouts

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

user, err := service.GetUser(ctx, 123)
```

## Defer for Cleanup

```go
f, err := os.Open("file.txt")
if err != nil {
    return err
}
defer f.Close()  // Always runs when function exits

// Continue working with f...
```

**Order:** Defers execute LIFO (last in, first out)

## Reference Documentation

For detailed information:
- **Standard library patterns:** See `@go-stdlib.md`
- **Tooling (go mod, test, fmt, lint):** See `@go-tooling.md`
- **Ecosystem libraries:** See `@go-ecosystem.md`

## Real-World Impact

Following these patterns results in:
- Code that other Go developers immediately understand
- Fewer bugs from error handling and nil panics
- Better performance from proper allocation patterns
- Easier testing with interfaces and testify
- Consistent tooling across team

**Bottom line:** Go rewards simplicity and explicitness. When in doubt, choose the simpler, more explicit approach.
