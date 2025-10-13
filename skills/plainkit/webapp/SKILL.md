---
name: PlainKit Web Application Architecture
description: Architecture patterns, conventions, and best practices for building Go web applications with PlainKit HTML and HTMX - fat services, thin handlers, hypermedia-driven interactions
when_to_use: When scaffolding, analyzing, or building Go web applications with PlainKit ecosystem (HTML + HTMX). For HTML basics see skills/plainkit/html, for HTMX attributes see this skill's HTMX section.
version: 2.0.0
---

# PlainKit Web Application Architecture

## Overview

Build production-ready Go web applications using PlainKit ecosystem: type-safe HTML generation with `plainkit/html` and hypermedia-driven interactivity with `plainkit/htmx`.

**Core principle:** _"Clear is better than clever."_ — Effective Go. Build simple, composable applications with fat services, thin handlers, and server-rendered HTML.

**Related skills:**

- skills/plainkit/html - Core HTML generation patterns
- skills/plainkit/svg - SVG elements
- skills/plainkit/icons - Lucide icons
- skills/testing/test-driven-development - TDD workflow for implementation

## Quick Reference

### Technology Stack

```go
import . "github.com/plainkit/html"  // Type-safe HTML
import . "github.com/plainkit/htmx"  // Hypermedia interactivity
import "net/http"                    // Standard library HTTP
```

### Repository Layout

```
myapp/
├── cmd/server/main.go      # Entry point
├── internal/
│   ├── app/                # Wiring, routes
│   ├── handlers/           # HTTP handlers (thin)
│   ├── service/            # Business logic (fat)
│   ├── store/              # Persistence interfaces + impls
│   ├── domain/             # Entities
│   ├── views/              # HTML rendering
│   ├── ui/                 # UI components
│   ├── css/                # Tailwind embed
│   └── middleware/         # HTTP middleware
├── testdata/
├── go.mod
├── .golangci.yml           # Linter config (mandatory)
└── Makefile
```

### Dependency Flow

```
domain → store → service → handlers → app → cmd/server/main.go
```

_Dependencies point inward_ — Clean Architecture principle

## Code Quality: Linting

**Every PlainKit webapp MUST use `golangci-lint` with `wsl_v5` for consistent formatting.**

### Setup: .golangci.yml

Create `.golangci.yml` in the root directory (where `go.mod` lives):

```yaml
version: "2"

linters:
  enable:
    - wsl_v5
  settings:
    wsl_v5:
      allow-first-in-block: true
      allow-whole-block: false
      branch-max-lines: 2
    staticcheck:
      checks:
        - "-ST1001" # Allow dot imports for plainkit/html
```

### Workflow: Run After Every Code Change

```bash
# Fix issues automatically
golangci-lint run --fix ./...

# Or add to Makefile
lint:
	golangci-lint run --fix ./...
```

**Why:** `wsl_v5` enforces consistent whitespace/line grouping, making code easier to read and maintain. Run after every code change, not just at the end.

## Go Interfaces: Best Practices

_"The bigger the interface, the weaker the abstraction."_ — Rob Pike

### 1. Keep Interfaces Small

**Bad:**

```go
type UserRepository interface {
    Create(user User) error
    Update(user User) error
    Delete(id string) error
    Get(id string) (User, error)
    List(limit, offset int) ([]User, error)
    FindByEmail(email string) (User, error)
    Count() (int, error)
}
```

**Good:**

```go
// Define only what you need
type UserGetter interface {
    Get(ctx context.Context, id string) (*domain.User, error)
}

type UserCreator interface {
    Create(ctx context.Context, user *domain.User) error
}

// Compose when needed
type UserStore interface {
    UserGetter
    UserCreator
    List(ctx context.Context, limit, offset int) ([]*domain.User, error)
}
```

### 2. Define Interfaces Where They're Used (Consumer)

**Bad:**

```go
// internal/store/user.go (producer defines interface)
type UserStore interface { ... }
type SqlUserStore struct { ... }
```

**Good:**

```go
// internal/service/user.go (consumer defines interface)
type userStore interface {
    Get(ctx context.Context, id string) (*domain.User, error)
    Create(ctx context.Context, user *domain.User) error
}

type UserService struct {
    store userStore  // Service defines what it needs
}

// internal/store/user.go (producer implements)
type GormUserStore struct { ... }
// Implicit satisfaction - no "implements" keyword
```

**Why:** Consumer knows what it needs. Producer might provide more. Decouples packages.

### 3. Accept Interfaces, Return Structs

_"Accept interfaces, return structs."_ — Go Proverb

```go
// Good: Accept interface
func NewUserService(store userStore) *UserService {
    return &UserService{store: store}
}

// Good: Return concrete type
func (s *UserService) Get(ctx context.Context, id string) (*domain.User, error) {
    return s.store.Get(ctx, id)
}
```

### 4. Use Composition Over Large Interfaces

```go
// Compose small interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}
```

## Architecture Layers

### Layer 1: Domain (Pure Data)

_"Make the zero value useful."_ — Effective Go

```go
// internal/domain/user.go
package domain

import "time"

type User struct {
    ID        string    `gorm:"primaryKey;type:varchar(36)"`
    Name      string    `gorm:"type:varchar(255);not null"`
    Email     string    `gorm:"type:varchar(255);uniqueIndex;not null"`
    CreatedAt time.Time `gorm:"autoCreateTime"`
    UpdatedAt time.Time `gorm:"autoUpdateTime"`
}

func (User) TableName() string { return "users" }
```

**Principles:**

- Pure data structures
- No business logic
- GORM tags for database mapping
- Zero value should be valid when possible

### Layer 2: Store (Persistence)

**Define interface in consumer (service), implement in store package.**

```go
// internal/service/user.go (interface defined by consumer)
type userStore interface {
    Get(ctx context.Context, id string) (*domain.User, error)
    Create(ctx context.Context, user *domain.User) error
    List(ctx context.Context, limit, offset int) ([]*domain.User, error)
}

// internal/store/gorm_user.go (implementation)
package store

type GormUserStore struct {
    db *gorm.DB
}

func NewGormUserStore(db *gorm.DB) *GormUserStore {
    return &GormUserStore{db: db}
}

func (s *GormUserStore) Create(ctx context.Context, user *domain.User) error {
    return s.db.WithContext(ctx).Create(user).Error
}

func (s *GormUserStore) Get(ctx context.Context, id string) (*domain.User, error) {
    var user domain.User
    result := s.db.WithContext(ctx).First(&user, "id = ?", id)

    if errors.Is(result.Error, gorm.ErrRecordNotFound) {
        return nil, errors.New("user not found")
    }

    return &user, result.Error
}
```

**Database initialization:**

```go
// internal/store/db.go
func NewDB(dbPath string, debug bool) (*gorm.DB, error) {
    config := &gorm.Config{
        Logger: logger.Default.LogMode(logger.Silent),
    }

    if debug {
        config.Logger = logger.Default.LogMode(logger.Info)
    }

    db, err := gorm.Open(sqlite.Open(dbPath), config)
    if err != nil {
        return nil, err
    }

    // Auto-migrate schemas
    if err := db.AutoMigrate(&domain.User{}); err != nil {
        return nil, err
    }

    return db, nil
}
```

**Dependencies (go.mod):**

```go
require (
    gorm.io/driver/sqlite v1.5.4
    gorm.io/gorm v1.25.5
    modernc.org/sqlite v1.27.0  // Pure Go (no CGO)
)
```

### Layer 3: Service (Business Logic - Fat)

_"A little copying is better than a little dependency."_ — Go Proverb

```go
// internal/service/user.go
package service

type CreateUserInput struct {
    Name  string
    Email string
}

// Define interface for store dependency (consumer-defined)
type userStore interface {
    Create(ctx context.Context, user *domain.User) error
    Get(ctx context.Context, id string) (*domain.User, error)
}

type UserService struct {
    store userStore
}

func NewUserService(store userStore) *UserService {
    return &UserService{store: store}
}

func (s *UserService) Create(ctx context.Context, input CreateUserInput) (*domain.User, error) {
    // Validation
    if strings.TrimSpace(input.Name) == "" {
        return nil, errors.New("name is required")
    }

    if !isValidEmail(input.Email) {
        return nil, errors.New("invalid email")
    }

    // Business logic
    user := &domain.User{
        ID:        uuid.NewString(),
        Name:      input.Name,
        Email:     input.Email,
        CreatedAt: time.Now(),
    }

    // Persistence
    if err := s.store.Create(ctx, user); err != nil {
        return nil, err
    }

    return user, nil
}
```

**Principles:**

- **Fat services** - All business logic here
- Input/output structs for type safety
- Validation happens in service layer
- Define minimal interfaces for dependencies
- Returns domain entities

### Layer 4: Handlers (HTTP Layer - Thin)

_"Errors are values."_ — Go Blog

```go
// internal/handlers/user.go
package handlers

type UserHandler struct {
    userService *service.UserService
}

func NewUserHandler(userService *service.UserService) *UserHandler {
    return &UserHandler{userService: userService}
}

func (h *UserHandler) ListUsers(w http.ResponseWriter, r *http.Request) {
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    offset, _ := strconv.Atoi(r.URL.Query().Get("offset"))

    users, err := h.userService.List(r.Context(), limit, offset)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    html := views.UsersPage(users)
    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    if err := r.ParseForm(); err != nil {
        http.Error(w, "Invalid form", http.StatusBadRequest)
        return
    }

    user, err := h.userService.Create(r.Context(), service.CreateUserInput{
        Name:  r.FormValue("name"),
        Email: r.FormValue("email"),
    })

    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // POST-Redirect-GET pattern
    http.Redirect(w, r, "/users/"+user.ID, http.StatusSeeOther)
}

// HTMX endpoint for infinite scroll
func (h *UserHandler) LoadMoreUsers(w http.ResponseWriter, r *http.Request) {
    offset, _ := strconv.Atoi(r.URL.Query().Get("offset"))

    users, err := h.userService.List(r.Context(), 10, offset)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Return HTML fragment for HTMX
    html := views.UserRows(users)
    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}
```

**Principles:**

- **Thin handlers** - Minimal logic
- Parse requests, delegate to service
- Return HTML (not JSON)
- Return HTML fragments for HTMX requests
- Use POST-Redirect-GET pattern

### Layer 5: Views (HTML Rendering)

```go
// internal/views/user.go
package views

import (
    . "github.com/plainkit/html"
    . "github.com/plainkit/htmx"
)

func UsersPage(users []*domain.User) string {
    content := Div(
        AClass("container mx-auto p-4"),
        H1(AClass("text-2xl font-bold mb-4"), T("Users")),

        Table(
            AClass("w-full"),
            Tbody(AId("user-tbody"), Fragment(UserRows(users)...)),
        ),

        Button(
            AClass("mt-4 px-4 py-2 bg-blue-500 text-white"),
            HxGet("/users/more?offset="+strconv.Itoa(len(users))),
            HxTarget("#user-tbody"),
            HxSwap("beforeend"),
            T("Load More"),
        ),
    )

    return Layout("Users", content)
}

// Fragment for HTMX updates
func UserRows(users []*domain.User) []Node {
    rows := make([]Node, 0, len(users))

    for _, user := range users {
        rows = append(rows, Tr(
            Td(T(user.Name)),
            Td(T(user.Email)),
        ))
    }

    return rows
}

func Layout(title string, content Node) string {
    page := Html(
        Head(
            Meta(ACharset("utf-8")),
            Title(T(title)),
            Link(ARel("stylesheet"), AHref("/css/output.css")),
            Script(ASrc("/js/htmx.min.js")),
        ),
        Body(content),
    )

    return Render(page)
}
```

### Layer 6: App (Wiring & Routes)

```go
// internal/app/app.go
package app

import (
    "net/http"

    "myapp/internal/handlers"
    "myapp/internal/service"
    "myapp/internal/store"
)

type App struct {
    mux    *http.ServeMux
    logger *slog.Logger
}

func New(logger *slog.Logger, db *gorm.DB) *App {
    // Wire dependencies
    userStore := store.NewGormUserStore(db)
    userService := service.NewUserService(userStore)
    userHandler := handlers.NewUserHandler(userService)

    // Setup routes
    mux := http.NewServeMux()
    mux.HandleFunc("GET /users", userHandler.ListUsers)
    mux.HandleFunc("POST /users", userHandler.CreateUser)
    mux.HandleFunc("GET /users/more", userHandler.LoadMoreUsers)

    return &App{mux: mux, logger: logger}
}

func (a *App) Handler() http.Handler {
    return middleware.Chain(
        a.mux,
        middleware.Logger(a.logger),
        middleware.Recovery(a.logger),
    )
}
```

### Layer 7: Main (Entry Point)

```go
// cmd/server/main.go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "time"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    db, err := store.NewDB("./data.db", true)
    if err != nil {
        logger.Error("database init failed", "error", err)
        os.Exit(1)
    }
    defer store.CloseDB(db)

    app := app.New(logger, db)

    server := &http.Server{
        Addr:         ":8080",
        Handler:      app.Handler(),
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Graceful shutdown
    go func() {
        logger.Info("server starting", "addr", server.Addr)
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            logger.Error("server error", "error", err)
            os.Exit(1)
        }
    }()

    sigint := make(chan os.Signal, 1)
    signal.Notify(sigint, os.Interrupt)
    <-sigint

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        logger.Error("shutdown error", "error", err)
    }
}
```

## HTMX Patterns

### Core Attributes

```go
import . "github.com/plainkit/htmx"

Button(
    HxGet("/load-more"),        // HTTP method + endpoint
    HxTarget("#list"),          // Target element selector
    HxSwap("beforeend"),        // Swap strategy
    HxTrigger("click"),         // Trigger event
)
```

**Swap Strategies:**

- `innerHTML` - Replace inner HTML (default)
- `outerHTML` - Replace entire element
- `beforeend` - Append to target
- `afterbegin` - Prepend to target
- `none` - Don't swap (side effects only)

### Pattern: Infinite Scroll

```go
// View returns rows + next load button
func ItemRows(items []Item, nextOffset int) string {
    rows := make([]Node, 0, len(items)+1)

    for _, item := range items {
        rows = append(rows, Div(T(item.Name)))
    }

    rows = append(rows, Button(
        AId("load-more"),
        HxGet("/items/more?offset="+strconv.Itoa(nextOffset)),
        HxTarget("#items"),
        HxSwap("beforeend"),
        T("Load More"),
    ))

    return Render(Fragment(rows...))
}
```

### Pattern: Inline Edit

```go
// Display mode
Div(
    AId("user-name"),
    Span(T(user.Name)),
    Button(
        HxGet("/users/"+user.ID+"/edit"),
        HxTarget("#user-name"),
        HxSwap("outerHTML"),
        T("Edit"),
    ),
)

// Edit mode (handler returns this)
Form(
    AId("user-name"),
    HxPost("/users/"+user.ID),
    HxTarget("#user-name"),
    HxSwap("outerHTML"),
    Input(AType("text"), AName("name"), AValue(user.Name)),
    Button(AType("submit"), T("Save")),
)
```

### Pattern: Live Form Validation

```go
// Input with validation
Input(
    AType("email"),
    AName("email"),
    HxGet("/validate/email"),
    HxTrigger("keyup changed delay:500ms"),
    HxTarget("#email-validation"),
)
Span(AId("email-validation"))

// Handler returns validation message
func (h *Handler) ValidateEmail(w http.ResponseWriter, r *http.Request) {
    email := r.URL.Query().Get("email")

    if !isValidEmail(email) {
        html := Render(Span(AClass("text-red-500"), T("Invalid email")))
        w.Write([]byte(html))
        return
    }

    html := Render(Span(AClass("text-green-500"), T("✓")))
    w.Write([]byte(html))
}
```

## Testing with TDD

**ALWAYS use Test-Driven Development when implementing features.**

See **skills/testing/test-driven-development** for complete TDD workflow.

### Quick TDD Pattern

1. **Write failing test** (RED)
2. **Write minimal code to pass** (GREEN)
3. **Refactor** while tests pass (REFACTOR)

### Service Tests

```go
func TestUserService_Create(t *testing.T) {
    // Arrange
    store := store.NewInMemoryUserStore()
    svc := service.NewUserService(store)

    // Act
    user, err := svc.Create(context.Background(), service.CreateUserInput{
        Name:  "John",
        Email: "john@example.com",
    })

    // Assert
    assert.NoError(t, err)
    assert.NotEmpty(t, user.ID)
    assert.Equal(t, "John", user.Name)
}
```

### Handler Tests

```go
func TestUserHandler_ListUsers(t *testing.T) {
    store := store.NewInMemoryUserStore()
    svc := service.NewUserService(store)
    handler := handlers.NewUserHandler(svc)

    req := httptest.NewRequest("GET", "/users", nil)
    w := httptest.NewRecorder()

    handler.ListUsers(w, req)

    assert.Equal(t, http.StatusOK, w.Code)
    assert.Contains(t, w.Body.String(), "<table")
}
```

## Development Workflow

### 1. Initial Setup

```bash
# Initialize project
go mod init myapp

# Install dependencies
go get github.com/plainkit/html
go get github.com/plainkit/htmx
go get gorm.io/gorm
go get gorm.io/driver/sqlite

# Create .golangci.yml (see "Code Quality: Linting" section)
# Create directory structure (see "Repository Layout")
```

### 2. Development Loop (TDD)

```bash
# 1. Write failing test
# 2. Run tests
go test ./...

# 3. Write minimal implementation
# 4. Run linter + fix
golangci-lint run --fix ./...

# 5. Tests pass → Refactor
# 6. Repeat
```

### 3. Running the App

```bash
# Build and run
go run cmd/server/main.go

# Or use air for live reload
air
```

### Makefile

```makefile
.PHONY: test lint run css

test:
	go test -v ./...

lint:
	golangci-lint run --fix ./...

run:
	go run cmd/server/main.go

css:
	tailwindcss -i ./internal/css/index.css -o ./internal/css/output.css --minify

dev:
	make -j2 watch-css run

watch-css:
	tailwindcss -i ./internal/css/index.css -o ./internal/css/output.css --watch
```

## Checklist: New PlainKit Webapp

### Setup

- [ ] Create `.golangci.yml` with `wsl_v5` linter
- [ ] Initialize `go.mod` with PlainKit dependencies
- [ ] Create directory structure (domain, store, service, handlers, views, app)
- [ ] Setup GORM with SQLite (`store/db.go`)
- [ ] Configure Tailwind CSS build

### Architecture

- [ ] Define domain models with GORM tags
- [ ] Define store interfaces **in service package** (consumer-defined)
- [ ] Implement GORM stores in `store/` package
- [ ] Write fat services with business logic
- [ ] Write thin handlers (parse + delegate)
- [ ] Create view functions returning HTML/fragments
- [ ] Wire dependencies in `app/` package
- [ ] Setup graceful shutdown in `main.go`

### Code Quality (Every Code Change)

- [ ] Write tests first (TDD - see skills/testing/test-driven-development)
- [ ] Run `golangci-lint run --fix ./...` after writing code
- [ ] Verify tests pass: `go test ./...`

### HTMX Integration

- [ ] Serve `htmx.min.js` from embedded JS
- [ ] Return HTML fragments for HTMX endpoints
- [ ] Use `HxGet`, `HxPost`, `HxTarget`, `HxSwap` attributes
- [ ] Implement POST-Redirect-GET for form submissions

### Best Practices

- [ ] Keep interfaces small (single method when possible)
- [ ] Define interfaces close to consumer, not producer
- [ ] Accept interfaces, return structs
- [ ] Use `context.Context` for cancellation
- [ ] Return HTML, not JSON (unless building API)
- [ ] Embed static assets (CSS, JS) for single binary

## Summary

PlainKit web applications follow these principles:

1. **Fat services, thin handlers** - Business logic in services
2. **Small, consumer-defined interfaces** - Define at call site
3. **Type-safe HTML** - Compile-time validation via plainkit/html
4. **Hypermedia-driven** - HTMX for interactivity
5. **GORM + SQLite3** - Pure Go database (no CGO)
6. **Consistent code quality** - `golangci-lint` with `wsl_v5` after every change
7. **TDD workflow** - Write tests first (see skills/testing/test-driven-development)
8. **Idiomatic Go** - Follow Effective Go principles
9. **Clear architecture** - Domain → Store → Service → Handler → App → Main

**Related skills:**

- skills/plainkit/html - HTML generation patterns
- skills/plainkit/svg - SVG elements
- skills/plainkit/icons - Lucide icons
- skills/testing/test-driven-development - TDD workflow

_"Simplicity is the ultimate sophistication."_ — Leonardo da Vinci
