---
name: PlainKit Web Application Architecture
description: Architecture patterns, conventions, and best practices for building Go web applications with PlainKit HTML and HTMX - fat services, thin handlers, hypermedia-driven interactions
when_to_use: When scaffolding, analyzing, or building Go web applications with PlainKit ecosystem (HTML + HTMX). For HTML basics see skills/plainkit/html, for HTMX attributes see this skill's HTMX section.
version: 1.0.0
---

# PlainKit Web Application Architecture

## Overview

This skill documents the architecture, patterns, and conventions for building production-ready Go web applications using the PlainKit ecosystem: type-safe HTML generation with `plainkit/html` and hypermedia-driven interactivity with `plainkit/htmx`.

**Core principle:** *"Clear is better than clever."* — Effective Go. Build simple, composable applications with fat services, thin handlers, and server-rendered HTML.

**Related skills:**
- skills/plainkit/html - Core HTML generation patterns
- skills/plainkit/svg - SVG elements (if needed)
- skills/plainkit/icons - Lucide icons

## Quick Reference

### Technology Stack

```go
// HTML rendering (type-safe)
import . "github.com/plainkit/html"

// HTMX integration (hypermedia interactivity)
import . "github.com/plainkit/htmx"

// Standard library HTTP
import "net/http"
```

### Repository Layout

```
myapp/
├── cmd/server/main.go      # Entry point
├── internal/
│   ├── app/                # Wiring, routes
│   ├── handlers/           # HTTP handlers (thin)
│   ├── service/            # Business logic (fat)
│   ├── store/              # Persistence
│   ├── domain/             # Entities
│   ├── views/              # HTML rendering
│   ├── ui/                 # UI components
│   ├── css/                # Tailwind embed
│   └── middleware/         # HTTP middleware
├── testdata/
├── go.mod
└── Makefile
```

### Dependency Flow

```
store → service → handlers → app → cmd/server/main.go
```

*"Dependencies point inward"* — Clean Architecture principle

## Core Technologies

### PlainKit HTML — Type-Safe HTML Builder

```go
import . "github.com/plainkit/html"

// Element creation
Div(
    AClass("container"),
    H1(T("Title")),
    P(T("Paragraph")),
)

// Text helpers
T("escaped text")           // HTML-escaped
UnsafeText("<b>raw</b>")   // Unescaped HTML
Fragment(node1, node2)      // Group nodes

// Attributes
AClass("class")
AId("id")
AHref("/path")
AType("submit")
```

**Key insight:** *"Don't communicate by sharing memory; share memory by communicating."* — Effective Go. Components compose through function calls, not shared state.

See skills/plainkit/html for complete documentation.

### PlainKit HTMX — Hypermedia Interactivity

```go
import . "github.com/plainkit/htmx"

// HTMX attributes
Button(
    HxGet("/load-more"),        // GET request
    HxPost("/submit"),          // POST request
    HxTarget("#list"),          // Target element
    HxSwap("beforeend"),        // Swap strategy
    HxTrigger("click"),         // Trigger event
    T("Load More"),
)
```

**HTMX Swap Strategies:**
- `innerHTML` - Replace inner HTML (default)
- `outerHTML` - Replace entire element
- `beforebegin` - Insert before target
- `afterbegin` - Insert at start of target
- `beforeend` - Insert at end of target
- `afterend` - Insert after target
- `delete` - Delete target
- `none` - Don't swap

**Serve HTMX JavaScript:**

```go
mux.HandleFunc("GET /js/htmx.min.js", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/javascript")
    w.Write(htmx.JavaScript())
})
```

## Architecture Layers

### Layer 1: Domain (Pure Data)

*"Make the zero value useful."* — Effective Go

```go
// internal/domain/user.go
package domain

import "time"

// User represents a user entity
type User struct {
    ID        string    `gorm:"primaryKey;type:varchar(36)"`
    Name      string    `gorm:"type:varchar(255);not null"`
    Email     string    `gorm:"type:varchar(255);uniqueIndex;not null"`
    CreatedAt time.Time `gorm:"autoCreateTime"`
    UpdatedAt time.Time `gorm:"autoUpdateTime"`
}

// TableName specifies the table name
func (User) TableName() string {
    return "users"
}

// No business logic, minimal dependencies - pure data with GORM tags
```

**Principles:**
- Pure data structures
- No business logic
- No dependencies on other layers
- Zero value should be valid when possible

### Layer 2: Store (Persistence)

*"Accept interfaces, return structs."* — Go Proverb

```go
// internal/store/user.go
package store

import (
    "context"
    "myapp/internal/domain"
)

// UserStore defines persistence operations
type UserStore interface {
    Create(ctx context.Context, user *domain.User) error
    Get(ctx context.Context, id string) (*domain.User, error)
    List(ctx context.Context, limit, offset int) ([]*domain.User, error)
    Update(ctx context.Context, user *domain.User) error
    Delete(ctx context.Context, id string) error
}

// InMemoryUserStore implements UserStore
type InMemoryUserStore struct {
    users map[string]*domain.User
    mu    sync.RWMutex
}

func NewInMemoryUserStore() *InMemoryUserStore {
    return &InMemoryUserStore{
        users: make(map[string]*domain.User),
    }
}

func (s *InMemoryUserStore) Create(ctx context.Context, user *domain.User) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, exists := s.users[user.ID]; exists {
        return errors.New("user already exists")
    }

    s.users[user.ID] = user
    return nil
}

func (s *InMemoryUserStore) Get(ctx context.Context, id string) (*domain.User, error) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    user, exists := s.users[id]
    if !exists {
        return nil, errors.New("user not found")
    }

    return user, nil
}
```

**Principles:**
- Define interface in store package
- Return concrete types (domain entities)
- Use context.Context for cancellation
- Thread-safe for in-memory implementations

#### Database Store Implementation (GORM + SQLite3)

**Recommended:** Use GORM with SQLite3 for simple, type-safe database operations with pure Go (no CGO required when using modernc.org/sqlite).

```go
// internal/store/gorm_user.go
package store

import (
    "context"
    "errors"

    "gorm.io/gorm"
    "myapp/internal/domain"
)

type GormUserStore struct {
    db *gorm.DB
}

func NewGormUserStore(db *gorm.DB) *GormUserStore {
    return &GormUserStore{db: db}
}

func (s *GormUserStore) Create(ctx context.Context, user *domain.User) error {
    result := s.db.WithContext(ctx).Create(user)
    return result.Error
}

func (s *GormUserStore) Get(ctx context.Context, id string) (*domain.User, error) {
    var user domain.User
    result := s.db.WithContext(ctx).First(&user, "id = ?", id)

    if errors.Is(result.Error, gorm.ErrRecordNotFound) {
        return nil, errors.New("user not found")
    }

    return &user, result.Error
}

func (s *GormUserStore) List(ctx context.Context, limit, offset int) ([]*domain.User, error) {
    var users []*domain.User
    result := s.db.WithContext(ctx).
        Order("created_at DESC").
        Limit(limit).
        Offset(offset).
        Find(&users)

    return users, result.Error
}

func (s *GormUserStore) Update(ctx context.Context, user *domain.User) error {
    result := s.db.WithContext(ctx).Save(user)
    return result.Error
}

func (s *GormUserStore) Delete(ctx context.Context, id string) error {
    result := s.db.WithContext(ctx).Delete(&domain.User{}, "id = ?", id)
    return result.Error
}
```

**Database initialization with GORM:**

```go
// internal/store/db.go
package store

import (
    "log"

    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"

    "myapp/internal/domain"
)

// NewDB initializes GORM with SQLite3 (pure Go, no CGO)
func NewDB(dbPath string, debug bool) (*gorm.DB, error) {
    var gormLogger logger.Interface
    if debug {
        gormLogger = logger.Default.LogMode(logger.Info)
    } else {
        gormLogger = logger.Default.LogMode(logger.Silent)
    }

    db, err := gorm.Open(sqlite.Open(dbPath), &gorm.Config{
        Logger: gormLogger,
    })
    if err != nil {
        return nil, err
    }

    // Auto-migrate schemas
    if err := db.AutoMigrate(
        &domain.User{},
        // Add other models here
    ); err != nil {
        return nil, err
    }

    return db, nil
}

// CloseDB closes the database connection
func CloseDB(db *gorm.DB) error {
    sqlDB, err := db.DB()
    if err != nil {
        return err
    }
    return sqlDB.Close()
}
```

**Domain model with GORM tags:**

```go
// internal/domain/user.go
package domain

import (
    "time"
)

type User struct {
    ID        string    `gorm:"primaryKey;type:varchar(36)"`
    Name      string    `gorm:"type:varchar(255);not null"`
    Email     string    `gorm:"type:varchar(255);uniqueIndex;not null"`
    CreatedAt time.Time `gorm:"autoCreateTime"`
    UpdatedAt time.Time `gorm:"autoUpdateTime"`
}

// TableName specifies the table name
func (User) TableName() string {
    return "users"
}
```

**Dependencies (go.mod):**

```go
require (
    gorm.io/driver/sqlite v1.5.4
    gorm.io/gorm v1.25.5
    modernc.org/sqlite v1.27.0 // Pure Go SQLite driver (no CGO)
)
```

**Why GORM + SQLite3:**
- **Pure Go**: modernc.org/sqlite requires no CGO (easier cross-compilation)
- **Type-safe**: Compile-time validation of queries
- **Auto-migration**: Schema management built-in
- **Simple**: Less boilerplate than database/sql
- **Portable**: Single-file database, perfect for development and small apps
- **Production-ready**: SQLite is battle-tested and fast for read-heavy workloads
```

### Layer 3: Service (Business Logic - Fat)

*"A little copying is better than a little dependency."* — Go Proverb

```go
// internal/service/user.go
package service

import (
    "context"
    "errors"
    "strings"
    "time"

    "github.com/google/uuid"
    "myapp/internal/domain"
    "myapp/internal/store"
)

// CreateUserInput is input for creating a user
type CreateUserInput struct {
    Name  string
    Email string
}

// UserService encapsulates user business logic
type UserService struct {
    store store.UserStore
}

func NewUserService(store store.UserStore) *UserService {
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

func (s *UserService) List(ctx context.Context, limit, offset int) ([]*domain.User, error) {
    if limit <= 0 || limit > 100 {
        limit = 20 // Default
    }

    return s.store.List(ctx, limit, offset)
}

func isValidEmail(email string) bool {
    return strings.Contains(email, "@")
}
```

**Principles:**
- **Fat services** - All business logic here
- Input/output structs for type safety
- Validation happens in service layer
- Returns domain entities
- No HTTP knowledge (uses context, not http.Request)

### Layer 4: Handlers (HTTP Layer - Thin)

*"Errors are values."* — Go Blog

```go
// internal/handlers/user.go
package handlers

import (
    "net/http"
    "strconv"

    "myapp/internal/service"
    "myapp/internal/views"
)

type UserHandler struct {
    userService *service.UserService
}

func NewUserHandler(userService *service.UserService) *UserHandler {
    return &UserHandler{userService: userService}
}

// ListUsers handles GET /users
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

// CreateUser handles POST /users
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

    // Redirect after POST (PRG pattern)
    http.Redirect(w, r, "/users/"+user.ID, http.StatusSeeOther)
}

// LoadMoreUsers handles HTMX GET /users/more
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
- Render views (HTML, not JSON)
- Return HTML fragments for HTMX
- Use POST-Redirect-GET pattern

### Layer 5: Views (HTML Rendering)

*"Simplicity is complicated."* — Rob Pike

```go
// internal/views/user.go
package views

import (
    . "github.com/plainkit/html"
    . "github.com/plainkit/htmx"

    "myapp/internal/domain"
)

// UsersPage renders the full users page
func UsersPage(users []*domain.User) string {
    content := Div(
        AClass("container mx-auto p-4"),
        H1(AClass("text-2xl font-bold mb-4"), T("Users")),

        // User table
        Table(
            AClass("w-full"),
            AId("user-table"),
            Thead(
                Tr(
                    Th(T("Name")),
                    Th(T("Email")),
                    Th(T("Created")),
                ),
            ),
            Tbody(
                AId("user-tbody"),
                Fragment(UserRows(users)...),
            ),
        ),

        // Load more button (HTMX)
        Button(
            AClass("mt-4 px-4 py-2 bg-blue-500 text-white rounded"),
            HxGet("/users/more?offset="+string(len(users))),
            HxTarget("#user-tbody"),
            HxSwap("beforeend"),
            T("Load More"),
        ),
    )

    return Layout("Users", content)
}

// UserRows renders table rows (for HTMX partial updates)
func UserRows(users []*domain.User) []Node {
    rows := make([]Node, 0, len(users))

    for _, user := range users {
        rows = append(rows, Tr(
            Td(T(user.Name)),
            Td(T(user.Email)),
            Td(T(user.CreatedAt.Format("2006-01-02"))),
        ))
    }

    return rows
}

// Layout wraps content in full HTML page
func Layout(title string, content Node) string {
    page := Html(
        Head(
            Meta(ACharset("utf-8")),
            Meta(AName("viewport"), AContent("width=device-width, initial-scale=1")),
            Title(T(title)),
            Link(ARel("stylesheet"), AHref("/css/output.css")),
            Script(ASrc("/js/htmx.min.js")),
        ),
        Body(
            content,
        ),
    )

    return Render(page)
}
```

**Principles:**
- Full pages for initial requests
- HTML fragments for HTMX updates
- Use Fragment() for collections
- Keep view logic minimal
- Return `string` (from Render()) or `[]Node` for composition

### Layer 6: App (Wiring & Routes)

*"Don't panic."* — Effective Go

```go
// internal/app/app.go
package app

import (
    "log/slog"
    "net/http"

    "myapp/internal/handlers"
    "myapp/internal/middleware"
    "myapp/internal/service"
    "myapp/internal/store"
)

type App struct {
    mux    *http.ServeMux
    logger *slog.Logger
}

func New(logger *slog.Logger, db *gorm.DB) *App {
    // Initialize stores (GORM-based)
    userStore := store.NewGormUserStore(db)

    // Initialize services
    userService := service.NewUserService(userStore)

    // Initialize handlers
    userHandler := handlers.NewUserHandler(userService)

    // Setup routes
    mux := http.NewServeMux()

    // Static assets
    mux.HandleFunc("GET /js/htmx.min.js", serveHTMX)
    mux.HandleFunc("GET /css/output.css", serveCSS)

    // User routes
    mux.HandleFunc("GET /users", userHandler.ListUsers)
    mux.HandleFunc("POST /users", userHandler.CreateUser)
    mux.HandleFunc("GET /users/more", userHandler.LoadMoreUsers)
    mux.HandleFunc("GET /users/{id}", userHandler.GetUser)

    return &App{
        mux:    mux,
        logger: logger,
    }
}

func (a *App) Handler() http.Handler {
    return middleware.Chain(
        a.mux,
        middleware.Logger(a.logger),
        middleware.Recovery(a.logger),
    )
}
```

**Principles:**
- Wire dependencies in New()
- Return http.Handler for testing
- Apply middleware via Chain()
- Use Go 1.22+ routing patterns

### Layer 7: Main (Entry Point)

*"Make the zero value useful."* — Effective Go

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

    "myapp/internal/app"
    "myapp/internal/store"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // Initialize GORM database (SQLite3)
    dbPath := os.Getenv("DB_PATH")
    if dbPath == "" {
        dbPath = "./data.db" // Default to local file
    }

    db, err := store.NewDB(dbPath, true) // true = debug mode
    if err != nil {
        logger.Error("Failed to connect to database", "error", err)
        os.Exit(1)
    }

    // Ensure database connection is closed on exit
    defer func() {
        if err := store.CloseDB(db); err != nil {
            logger.Error("Failed to close database", "error", err)
        }
    }()

    logger.Info("database initialized", "path", dbPath)

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

    // Wait for interrupt signal
    sigint := make(chan os.Signal, 1)
    signal.Notify(sigint, os.Interrupt)
    <-sigint

    logger.Info("shutting down server")

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        logger.Error("shutdown error", "error", err)
    }

    logger.Info("server stopped")
}
```

**Principles:**
- Graceful shutdown
- Structured logging
- Reasonable timeouts
- Single responsibility (bootstrap only)

## Configuration Management

*"The bigger the interface, the weaker the abstraction."* — Rob Pike

### Environment-Based Configuration

```go
// internal/config/config.go
package config

import (
    "errors"
    "os"
    "strconv"
    "time"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Logging  LoggingConfig
}

type ServerConfig struct {
    Port         string
    ReadTimeout  time.Duration
    WriteTimeout time.Duration
    IdleTimeout  time.Duration
}

type DatabaseConfig struct {
    Path  string // SQLite database file path
    Debug bool   // Enable GORM debug logging
}

type LoggingConfig struct {
    Level  string
    Format string // json or text
}

// Load reads configuration from environment variables
func Load() (*Config, error) {
    cfg := &Config{
        Server: ServerConfig{
            Port:         getEnv("PORT", "8080"),
            ReadTimeout:  getDuration("SERVER_READ_TIMEOUT", 15*time.Second),
            WriteTimeout: getDuration("SERVER_WRITE_TIMEOUT", 15*time.Second),
            IdleTimeout:  getDuration("SERVER_IDLE_TIMEOUT", 60*time.Second),
        },
        Database: DatabaseConfig{
            Path:  getEnv("DB_PATH", "./data.db"),
            Debug: getEnvBool("DB_DEBUG", false),
        },
        Logging: LoggingConfig{
            Level:  getEnv("LOG_LEVEL", "info"),
            Format: getEnv("LOG_FORMAT", "json"),
        },
    }

    if err := cfg.Validate(); err != nil {
        return nil, err
    }

    return cfg, nil
}

func (c *Config) Validate() error {
    if c.Database.Path == "" {
        return errors.New("DB_PATH is required")
    }
    return nil
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intVal, err := strconv.Atoi(value); err == nil {
            return intVal
        }
    }
    return defaultValue
}

func getDuration(key string, defaultValue time.Duration) time.Duration {
    if value := os.Getenv(key); value != "" {
        if duration, err := time.ParseDuration(value); err == nil {
            return duration
        }
    }
    return defaultValue
}

func getEnvBool(key string, defaultValue bool) bool {
    if value := os.Getenv(key); value != "" {
        return value == "true" || value == "1"
    }
    return defaultValue
}
```

### Using Configuration

```go
// cmd/server/main.go
func main() {
    cfg, err := config.Load()
    if err != nil {
        log.Fatal("Failed to load config:", err)
    }

    logger := setupLogger(cfg.Logging)

    db, err := store.NewDB(cfg.Database.Path, cfg.Database.Debug)
    if err != nil {
        logger.Error("Failed to connect to database", "error", err)
        os.Exit(1)
    }
    defer store.CloseDB(db)

    app := app.New(logger, db)

    server := &http.Server{
        Addr:         ":" + cfg.Server.Port,
        Handler:      app.Handler(),
        ReadTimeout:  cfg.Server.ReadTimeout,
        WriteTimeout: cfg.Server.WriteTimeout,
        IdleTimeout:  cfg.Server.IdleTimeout,
    }

    // ... graceful shutdown ...
}
```

### .env File Support

```go
// internal/config/dotenv.go
package config

import (
    "bufio"
    "os"
    "strings"
)

// LoadEnv loads environment variables from .env file
func LoadEnv(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err // It's okay if .env doesn't exist
    }
    defer file.Close()

    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        line := strings.TrimSpace(scanner.Text())

        // Skip comments and empty lines
        if line == "" || strings.HasPrefix(line, "#") {
            continue
        }

        // Split on first =
        parts := strings.SplitN(line, "=", 2)
        if len(parts) != 2 {
            continue
        }

        key := strings.TrimSpace(parts[0])
        value := strings.TrimSpace(parts[1])

        // Only set if not already set
        if os.Getenv(key) == "" {
            os.Setenv(key, value)
        }
    }

    return scanner.Err()
}
```

**Usage:**

```go
func main() {
    // Load .env in development
    if err := config.LoadEnv(".env"); err != nil {
        // Ignore error - .env is optional
    }

    cfg, err := config.Load()
    // ...
}
```

**Example .env file:**

```bash
# Server configuration
PORT=8080
SERVER_READ_TIMEOUT=15s
SERVER_WRITE_TIMEOUT=15s
SERVER_IDLE_TIMEOUT=60s

# Database configuration (SQLite)
DB_PATH=./data.db
DB_DEBUG=true

# Logging
LOG_LEVEL=info
LOG_FORMAT=json
```

## Error Handling Patterns

*"Errors are values."* — Rob Pike

### Error Types

```go
// internal/errors/errors.go
package errors

import (
    "errors"
    "fmt"
)

// Common error types
var (
    ErrNotFound      = errors.New("resource not found")
    ErrUnauthorized  = errors.New("unauthorized")
    ErrForbidden     = errors.New("forbidden")
    ErrBadRequest    = errors.New("bad request")
    ErrConflict      = errors.New("resource conflict")
    ErrInternal      = errors.New("internal server error")
)

// ValidationError represents validation failures
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// MultiError aggregates multiple errors
type MultiError struct {
    Errors []error
}

func (e *MultiError) Error() string {
    if len(e.Errors) == 0 {
        return "no errors"
    }
    if len(e.Errors) == 1 {
        return e.Errors[0].Error()
    }
    return fmt.Sprintf("%d errors occurred", len(e.Errors))
}

func (e *MultiError) Add(err error) {
    if err != nil {
        e.Errors = append(e.Errors, err)
    }
}

func (e *MultiError) HasErrors() bool {
    return len(e.Errors) > 0
}
```

### HTTP Error Responses

```go
// internal/handlers/errors.go
package handlers

import (
    "errors"
    "log/slog"
    "net/http"

    . "github.com/plainkit/html"
    apperrors "myapp/internal/errors"
)

// ErrorResponse renders error HTML for HTMX or full page
func ErrorResponse(w http.ResponseWriter, r *http.Request, err error, statusCode int) {
    w.Header().Set("Content-Type", "text/html")
    w.WriteHeader(statusCode)

    // For HTMX requests, return fragment
    if r.Header.Get("HX-Request") == "true" {
        html := Render(Div(
            AClass("bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded"),
            T(err.Error()),
        ))
        w.Write([]byte(html))
        return
    }

    // For full page requests, return complete error page
    html := Render(Html(
        Head(Title(T("Error"))),
        Body(
            H1(T("Error")),
            P(T(err.Error())),
            A(AHref("/"), T("Go Home")),
        ),
    ))
    w.Write([]byte(html))
}

// HandleError maps application errors to HTTP errors
func HandleError(w http.ResponseWriter, r *http.Request, logger *slog.Logger, err error) {
    switch {
    case errors.Is(err, apperrors.ErrNotFound):
        ErrorResponse(w, r, err, http.StatusNotFound)
    case errors.Is(err, apperrors.ErrUnauthorized):
        ErrorResponse(w, r, err, http.StatusUnauthorized)
    case errors.Is(err, apperrors.ErrForbidden):
        ErrorResponse(w, r, err, http.StatusForbidden)
    case errors.Is(err, apperrors.ErrBadRequest):
        ErrorResponse(w, r, err, http.StatusBadRequest)
    case errors.Is(err, apperrors.ErrConflict):
        ErrorResponse(w, r, err, http.StatusConflict)
    default:
        logger.Error("internal error", "error", err, "path", r.URL.Path)
        ErrorResponse(w, r, apperrors.ErrInternal, http.StatusInternalServerError)
    }
}
```

### Service Layer Error Wrapping

```go
// internal/service/user.go
func (s *UserService) Get(ctx context.Context, id string) (*domain.User, error) {
    user, err := s.store.Get(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("%w: user %s", apperrors.ErrNotFound, id)
        }
        return nil, fmt.Errorf("get user: %w", err)
    }

    return user, nil
}
```

### Handler Error Handling

```go
// internal/handlers/user.go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    user, err := h.userService.Get(r.Context(), id)
    if err != nil {
        HandleError(w, r, h.logger, err)
        return
    }

    html := views.UserPage(user)
    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}
```

## Middleware Patterns

### Authentication Middleware

```go
// internal/middleware/auth.go
package middleware

import (
    "context"
    "net/http"

    apperrors "myapp/internal/errors"
)

type contextKey string

const UserContextKey contextKey = "user"

// Auth middleware validates user session
func Auth(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Get session cookie
            cookie, err := r.Cookie("session_id")
            if err != nil {
                http.Redirect(w, r, "/login", http.StatusSeeOther)
                return
            }

            // Validate session (example - use real session store)
            userID := validateSession(cookie.Value)
            if userID == "" {
                http.Redirect(w, r, "/login", http.StatusSeeOther)
                return
            }

            // Add user to context
            ctx := context.WithValue(r.Context(), UserContextKey, userID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// RequireAuth protects specific routes
func RequireAuth(handler http.HandlerFunc, logger *slog.Logger) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        userID, ok := r.Context().Value(UserContextKey).(string)
        if !ok || userID == "" {
            http.Redirect(w, r, "/login", http.StatusSeeOther)
            return
        }

        handler(w, r)
    }
}
```

### Using Auth Middleware

```go
// internal/app/app.go
func New(logger *slog.Logger, db *sql.DB) *App {
    // ... setup stores and services ...

    mux := http.NewServeMux()

    // Public routes
    mux.HandleFunc("GET /login", authHandler.LoginPage)
    mux.HandleFunc("POST /login", authHandler.Login)

    // Protected routes
    mux.HandleFunc("GET /users", middleware.RequireAuth(userHandler.ListUsers, logger))
    mux.HandleFunc("POST /users", middleware.RequireAuth(userHandler.CreateUser, logger))

    return &App{mux: mux, logger: logger}
}
```

## HTMX Patterns

### Pattern 1: Infinite Scroll

```go
// Handler
func (h *Handler) LoadMore(w http.ResponseWriter, r *http.Request) {
    offset, _ := strconv.Atoi(r.URL.Query().Get("offset"))
    items, err := h.service.List(r.Context(), 10, offset)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    html := views.ItemRows(items, offset+10)
    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

// View
func ItemRows(items []Item, nextOffset int) string {
    rows := make([]Node, 0, len(items)+1)

    for _, item := range items {
        rows = append(rows, Div(T(item.Name)))
    }

    // Next load more button
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

### Pattern 2: Form Validation

```go
// Handler
func (h *Handler) ValidateEmail(w http.ResponseWriter, r *http.Request) {
    email := r.URL.Query().Get("email")

    if !isValidEmail(email) {
        html := Render(Span(
            AClass("text-red-500"),
            T("Invalid email"),
        ))
        w.Write([]byte(html))
        return
    }

    html := Render(Span(
        AClass("text-green-500"),
        T("✓ Valid"),
    ))
    w.Write([]byte(html))
}

// View - Input with live validation
Input(
    AType("email"),
    AName("email"),
    HxGet("/validate/email"),
    HxTrigger("keyup changed delay:500ms"),
    HxTarget("#email-validation"),
)
Span(AId("email-validation"))
```

### Pattern 3: Inline Edit

```go
// View - Display mode
func UserNameDisplay(user *domain.User) Node {
    return Div(
        AId("user-name"),
        Span(T(user.Name)),
        Button(
            HxGet("/users/"+user.ID+"/edit"),
            HxTarget("#user-name"),
            HxSwap("outerHTML"),
            T("Edit"),
        ),
    )
}

// View - Edit mode
func UserNameEdit(user *domain.User) Node {
    return Form(
        AId("user-name"),
        HxPost("/users/"+user.ID),
        HxTarget("#user-name"),
        HxSwap("outerHTML"),
        Input(
            AType("text"),
            AName("name"),
            AValue(user.Name),
        ),
        Button(AType("submit"), T("Save")),
        Button(
            AType("button"),
            HxGet("/users/"+user.ID),
            HxTarget("#user-name"),
            HxSwap("outerHTML"),
            T("Cancel"),
        ),
    )
}
```

## CSS Integration (Tailwind)

### Embed CSS for Single Binary

```go
// internal/css/embed.go
package css

import _ "embed"

//go:embed output.css
var CSS []byte
```

### Serve Embedded CSS

```go
func serveCSS(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/css")
    w.Write(css.CSS)
}
```

### Tailwind Configuration

```javascript
// tailwind.config.js
module.exports = {
  content: ["./internal/**/*.go"],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

### Build CSS

```makefile
# Makefile
css:
	tailwindcss -i ./internal/css/index.css -o ./internal/css/output.css --minify

watch-css:
	tailwindcss -i ./internal/css/index.css -o ./internal/css/output.css --watch

dev:
	make -j2 watch-css run
```

## Testing Patterns

### Service Tests

```go
func TestUserService_Create(t *testing.T) {
    store := store.NewInMemoryUserStore()
    svc := service.NewUserService(store)

    user, err := svc.Create(context.Background(), service.CreateUserInput{
        Name:  "John Doe",
        Email: "john@example.com",
    })

    assert.NoError(t, err)
    assert.NotEmpty(t, user.ID)
    assert.Equal(t, "John Doe", user.Name)
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

## Common Patterns Checklist

### Architecture
- [ ] Use **fat services, thin handlers**
- [ ] Define interfaces in consumer package (store)
- [ ] Accept interfaces, return structs
- [ ] Dependencies point inward (domain ← store ← service ← handlers)

### HTTP & Rendering
- [ ] Return HTML, not JSON (unless API required)
- [ ] Use HTMX for interactivity (avoid JavaScript)
- [ ] Use POST-Redirect-GET pattern
- [ ] Return HTML fragments for HTMX requests
- [ ] Embed static assets for single binary
- [ ] Use Go 1.22+ routing patterns

### Configuration & Environment
- [ ] Load configuration from environment variables
- [ ] Validate configuration on startup
- [ ] Support .env files for local development
- [ ] Never hardcode secrets or connection strings

### Error Handling
- [ ] Use context.Context for cancellation
- [ ] Wrap errors with context using fmt.Errorf
- [ ] Map domain errors to HTTP status codes
- [ ] Return HTML error responses (not JSON)
- [ ] Log internal errors with structured logging
- [ ] Validate in service layer, not handlers

### Database & Persistence (GORM + SQLite3)
- [ ] Use GORM with SQLite3 for type-safe queries
- [ ] Define domain models with GORM tags
- [ ] Use AutoMigrate for schema management
- [ ] Always pass context with WithContext(ctx)
- [ ] Handle gorm.ErrRecordNotFound explicitly
- [ ] Use modernc.org/sqlite for pure Go (no CGO)

### Security & Auth
- [ ] Use middleware for authentication
- [ ] Store user context in request context
- [ ] Validate sessions before accessing protected routes
- [ ] Use HTTPS in production

### Operations
- [ ] Implement graceful shutdown
- [ ] Use structured logging (slog)
- [ ] Set reasonable timeouts on http.Server
- [ ] Handle signals for clean shutdown

## Effective Go Principles

*"Gofmt's style is no one's favorite, yet gofmt is everyone's favorite."* — Rob Pike

1. **Formatting** - Use gofmt, no exceptions
2. **Commentary** - Comments should explain *why*, not *what*
3. **Names** - Short, meaningful, no underscores
4. **Control structures** - if/for/switch, no while/do
5. **Functions** - Multiple return values, named results
6. **Data** - Composition over inheritance
7. **Interfaces** - Small interfaces, implicit satisfaction
8. **Concurrency** - Goroutines and channels when needed
9. **Errors** - Errors are values, handle them
10. **Simplicity** - *"Less is more"*

## Summary

PlainKit web applications follow clean architecture with:

1. **Fat services, thin handlers** - Business logic in services
2. **Type-safe HTML** - Compile-time validation via plainkit/html
3. **Hypermedia-driven** - HTMX for interactivity, not JSON APIs
4. **Database integration** - GORM with SQLite3 (pure Go, no CGO)
5. **Configuration management** - Environment variables with .env support
6. **Error handling** - Typed errors with context wrapping
7. **Authentication** - Middleware-based auth with context propagation
8. **Single binary** - Embed CSS, JS, and SQLite database
9. **Idiomatic Go** - Follow Effective Go principles
10. **Clear structure** - Domain → Store → Service → Handler → App → Main

**What this skill covers:**
- 7-layer architecture with dependency injection
- Database store implementations (GORM with SQLite3, auto-migration)
- Configuration loading from environment variables
- HTTP error handling with HTMX-aware responses
- Authentication middleware patterns
- HTMX interaction patterns (infinite scroll, inline edit, validation)
- Testing patterns for services and handlers

**Related skills:**
- skills/plainkit/html - HTML generation patterns
- skills/plainkit/svg - SVG elements
- skills/plainkit/icons - Lucide icons

*"The key to performance is elegance, not battalions of special cases."* — Jon Bentley and Doug McIlroy
