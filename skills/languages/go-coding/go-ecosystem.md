# Go Ecosystem Libraries

Comprehensive guide to recommended libraries for Go development.

## CLI Applications

### Cobra (Command-line Interface)

**Package:** `github.com/spf13/cobra`

**Use for:** All CLI applications

```go
package cmd

import "github.com/spf13/cobra"

var rootCmd = &cobra.Command{
    Use:   "myapp",
    Short: "My application description",
}

var addCmd = &cobra.Command{
    Use:   "add [item]",
    Short: "Add an item",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        // Implementation
        return nil
    },
}

func init() {
    rootCmd.AddCommand(addCmd)
}

func Execute() error {
    return rootCmd.Execute()
}
```

**Key Features:**

- Subcommands (git-style: `app command subcommand`)
- Flags (persistent and local)
- Automatic help generation
- Shell completion
- `RunE` for error propagation

**Project Structure:**

```
myapp/
├── main.go              # Entry: cmd.Execute()
├── cmd/
│   ├── root.go         # Root command
│   ├── add.go          # Add subcommand
│   └── list.go         # List subcommand
└── internal/           # Business logic
```

## Configuration

### godotenv (Environment Variables)

**Package:** `github.com/joho/godotenv`

**Use for:** Loading .env files

```go
import (
    "os"
    "github.com/joho/godotenv"
)

func init() {
    // Load .env file (ignores error if missing)
    _ = godotenv.Load()
}

func main() {
    dbHost := os.Getenv("DB_HOST")
    dbPort := os.Getenv("DB_PORT")
}
```

**.env file:**

```bash
DB_HOST=localhost
DB_PORT=5432
API_KEY=secret123
```

**Best practices:**

- Never commit .env files (add to .gitignore)
- Provide .env.example with dummy values
- Use `godotenv.Load()` in main.go or init()
- Env vars override .env values

### Viper (Complex Configuration)

**Package:** `github.com/spf13/viper`

**Use for:** Multi-source config (files, env vars, flags)

```go
import "github.com/spf13/viper"

func LoadConfig() error {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    viper.AutomaticEnv()

    if err := viper.ReadInConfig(); err != nil {
        return err
    }
    return nil
}

// Access config
dbHost := viper.GetString("database.host")
port := viper.GetInt("server.port")
```

### envconfig (Struct Binding)

**Package:** `github.com/kelseyhightower/envconfig`

**Use for:** Type-safe environment variable parsing

```go
import "github.com/kelseyhightower/envconfig"

type Config struct {
    DatabaseURL  string `envconfig:"DATABASE_URL" required:"true"`
    Port         int    `envconfig:"PORT" default:"8080"`
    Debug        bool   `envconfig:"DEBUG" default:"false"`
}

func LoadConfig() (*Config, error) {
    var cfg Config
    if err := envconfig.Process("", &cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

## HTTP Servers

### net/http (Standard Library)

**Use for:** Small projects, simple routing

```go
import "net/http"

func main() {
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/api/users", usersHandler)
    http.ListenAndServe(":8080", nil)
}

func homeHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("Hello"))
}
```

**Middleware pattern:**

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

http.ListenAndServe(":8080", loggingMiddleware(http.DefaultServeMux))
```

### Gin (Production Framework)

**Package:** `github.com/gin-gonic/gin`

**Use for:** Production APIs, complex routing, middleware

```go
import "github.com/gin-gonic/gin"

func main() {
    router := gin.Default() // Includes logger + recovery middleware

    // Routes
    router.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "pong"})
    })

    // Route groups
    api := router.Group("/api/v1")
    {
        api.GET("/users", getUsers)
        api.POST("/users", createUser)
        api.GET("/users/:id", getUser)
    }

    router.Run(":8080")
}

func getUser(c *gin.Context) {
    id := c.Param("id")
    c.JSON(200, gin.H{"id": id, "name": "Alice"})
}
```

**Middleware:**

```go
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
            return
        }
        c.Next()
    }
}

api.Use(AuthMiddleware())
```

**Request binding:**

```go
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
}

func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    // Process request
}
```

## Database

### GORM (ORM)

**Package:** `gorm.io/gorm`

**Use for:** All database operations

```go
import (
    "gorm.io/driver/postgres"
    "gorm.io/driver/sqlite"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

// Connect
dsn := "host=localhost user=postgres password=secret dbname=mydb"
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})

// Models
type User struct {
    ID        uint           `gorm:"primarykey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"` // Soft delete
    Name      string         `gorm:"not null"`
    Email     string         `gorm:"uniqueIndex"`
}

// Auto migrate
db.AutoMigrate(&User{})

// Create
user := User{Name: "Alice", Email: "alice@example.com"}
db.Create(&user)

// Read
var user User
db.First(&user, 1) // By primary key
db.Where("email = ?", "alice@example.com").First(&user)

// Update
db.Model(&user).Update("Name", "Bob")
db.Model(&user).Updates(User{Name: "Bob", Email: "bob@example.com"})

// Delete (soft delete)
db.Delete(&user, 1)

// Permanent delete
db.Unscoped().Delete(&user, 1)
```

**Associations:**

```go
type User struct {
    gorm.Model
    Name    string
    Profile Profile
}

type Profile struct {
    gorm.Model
    UserID uint
    Bio    string
}

// Preload relationships
var user User
db.Preload("Profile").First(&user, 1)
```

**Transactions:**

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err
    }
    if err := tx.Create(&profile).Error; err != nil {
        return err
    }
    return nil
})
```

**Best Practices:**

- Use `gorm.Model` for standard fields (ID, CreatedAt, UpdatedAt, DeletedAt)
- Always check `.Error` on operations
- Use transactions for multi-step operations
- Use `Preload` to avoid N+1 queries
- Use indexes on frequently queried fields

## Testing

### testify (Assertions and Mocking)

**Package:** `github.com/stretchr/testify`

**Use for:** All tests (replaces stdlib testing assertions)

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestUserService(t *testing.T) {
    service := NewUserService()

    user, err := service.GetUser(1)

    // require: stops test on failure
    require.NoError(t, err)
    require.NotNil(t, user)

    // assert: continues on failure
    assert.Equal(t, "Alice", user.Name)
    assert.Equal(t, 25, user.Age)
}
```

**Table-driven tests:**

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            assert.Equal(t, tt.expected, result)
        })
    }
}
```

**Test suites:**

```go
import "github.com/stretchr/testify/suite"

type UserServiceSuite struct {
    suite.Suite
    db      *gorm.DB
    service *UserService
}

func (s *UserServiceSuite) SetupTest() {
    s.db = setupTestDB()
    s.service = NewUserService(s.db)
}

func (s *UserServiceSuite) TearDownTest() {
    s.db.Migrator().DropTable(&User{})
}

func (s *UserServiceSuite) TestCreateUser() {
    user, err := s.service.CreateUser("Alice", "alice@example.com")
    s.NoError(err)
    s.Equal("Alice", user.Name)
}

func TestUserServiceSuite(t *testing.T) {
    suite.Run(t, new(UserServiceSuite))
}
```

### moq (Mock Generation)

**Package:** `github.com/matryer/moq`

**Use for:** Generating mocks from interfaces

```go
//go:generate moq -out user_repo_mock.go . UserRepository

type UserRepository interface {
    GetByID(ctx context.Context, id int) (*User, error)
    Create(ctx context.Context, user *User) error
}
```

**Run generation:**

```bash
go generate ./...
```

**Use in tests:**

```go
func TestUserService_GetUser(t *testing.T) {
    mockRepo := &UserRepositoryMock{
        GetByIDFunc: func(ctx context.Context, id int) (*User, error) {
            return &User{ID: id, Name: "Alice"}, nil
        },
    }

    service := NewUserService(mockRepo)
    user, err := service.GetUser(context.Background(), 1)

    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
    assert.Equal(t, 1, len(mockRepo.GetByIDCalls()))
}
```

### httptest (HTTP Testing)

**Package:** `net/http/httptest` (stdlib)

**Use for:** Testing HTTP handlers

```go
import "net/http/httptest"

func TestHealthHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/health", nil)
    w := httptest.NewRecorder()

    healthHandler(w, req)

    assert.Equal(t, 200, w.Code)
    assert.Equal(t, "OK", w.Body.String())
}
```

**Testing Gin handlers:**

```go
func TestGetUser(t *testing.T) {
    gin.SetMode(gin.TestMode)

    router := gin.New()
    router.GET("/users/:id", getUser)

    req := httptest.NewRequest("GET", "/users/1", nil)
    w := httptest.NewRecorder()

    router.ServeHTTP(w, req)

    assert.Equal(t, 200, w.Code)
}
```

## Logging

### slog (Standard Library)

**Package:** `log/slog` (Go 1.21+)

**Use for:** Most applications

```go
import "log/slog"

// Configure
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
slog.SetDefault(logger)

// Log
slog.Info("user created", "id", user.ID, "name", user.Name)
slog.Error("failed to save", "error", err)

// With context
logger := slog.With("request_id", "abc123")
logger.Info("processing request")
```

### zerolog (High Performance)

**Package:** `github.com/rs/zerolog`

**Use for:** High-throughput applications

```go
import "github.com/rs/zerolog/log"

log.Info().
    Str("name", "Alice").
    Int("age", 25).
    Msg("user created")
```

### zap (Production Logging)

**Package:** `go.uber.org/zap`

**Use for:** Production systems requiring performance + structure

```go
import "go.uber.org/zap"

logger, _ := zap.NewProduction()
defer logger.Sync()

logger.Info("user created",
    zap.Int("id", user.ID),
    zap.String("name", user.Name),
)
```

## Validation

### validator (Struct Validation)

**Package:** `github.com/go-playground/validator/v10`

```go
import "github.com/go-playground/validator/v10"

type User struct {
    Name  string `validate:"required,min=2,max=50"`
    Email string `validate:"required,email"`
    Age   int    `validate:"gte=0,lte=130"`
}

validate := validator.New()

user := &User{Name: "A", Email: "invalid", Age: 200}
err := validate.Struct(user)

if err != nil {
    for _, err := range err.(validator.ValidationErrors) {
        fmt.Printf("Field: %s, Error: %s\n", err.Field(), err.Tag())
    }
}
```

**Custom validation:**

```go
validate.RegisterValidation("username", func(fl validator.FieldLevel) bool {
    username := fl.Field().String()
    return len(username) >= 3 && len(username) <= 20
})
```

## Time and Date

### carbon (Time Helper)

**Package:** `github.com/golang-module/carbon/v2`

**Use for:** Readable time operations

```go
import "github.com/golang-module/carbon/v2"

now := carbon.Now()
tomorrow := now.AddDay()
lastWeek := now.SubWeek()

formatted := now.ToDateTimeString() // 2025-10-26 10:00:00
```

## JSON

**Use stdlib:** `encoding/json`

```go
import "encoding/json"

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"` // Omit if empty
}

// Marshal
data, err := json.Marshal(user)

// Unmarshal
var user User
err := json.Unmarshal(data, &user)
```

## Summary

| Category   | Library       | When to Use          |
| ---------- | ------------- | -------------------- |
| CLI        | cobra         | All CLI apps         |
| Config     | godotenv      | .env files           |
| Config     | viper         | Complex config       |
| Config     | envconfig     | Type-safe env vars   |
| HTTP       | net/http      | Small projects       |
| HTTP       | gin           | Production APIs      |
| Database   | gorm          | All database work    |
| Testing    | testify       | All tests            |
| Mocking    | moq           | Mock generation      |
| Logging    | slog          | Most apps (Go 1.21+) |
| Logging    | zerolog/zap   | High performance     |
| Validation | validator/v10 | Struct validation    |

**Bottom line:** These libraries are production-tested and widely adopted. Use them instead of searching for alternatives.
