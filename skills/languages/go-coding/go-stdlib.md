# Go Standard Library Patterns

Common patterns and idioms for Go standard library packages.

## fmt (Formatting)

### String Formatting

```go
import "fmt"

// Basic printing
fmt.Println("Hello, World!")              // With newline
fmt.Print("No newline")                   // Without newline
fmt.Printf("Name: %s, Age: %d\n", name, age)  // Formatted

// String formatting (no printing)
s := fmt.Sprintf("User: %s", username)

// To writer
fmt.Fprintf(w, "Status: %d\n", 200)
```

### Common Verbs

| Verb | Type | Example |
|------|------|---------|
| `%v` | Default format | `fmt.Printf("%v", user)` |
| `%+v` | Struct with field names | `fmt.Printf("%+v", user)` |
| `%#v` | Go syntax | `fmt.Printf("%#v", user)` |
| `%T` | Type | `fmt.Printf("%T", user)` |
| `%s` | String | `fmt.Printf("%s", name)` |
| `%d` | Integer | `fmt.Printf("%d", count)` |
| `%f` | Float | `fmt.Printf("%.2f", price)` |
| `%t` | Boolean | `fmt.Printf("%t", active)` |
| `%p` | Pointer | `fmt.Printf("%p", &user)` |
| `%q` | Quoted string | `fmt.Printf("%q", str)` |
| `%w` | Wrap error | `fmt.Errorf("failed: %w", err)` |

## strings (String Manipulation)

```go
import "strings"

// Common operations
strings.Contains("hello", "ll")           // true
strings.HasPrefix("hello", "he")          // true
strings.HasSuffix("hello", "lo")          // true
strings.Split("a,b,c", ",")               // []string{"a", "b", "c"}
strings.Join([]string{"a", "b"}, ",")     // "a,b"
strings.Replace("hello", "l", "x", 2)     // "hexxo"
strings.ReplaceAll("hello", "l", "x")     // "hexxo"
strings.ToLower("HELLO")                  // "hello"
strings.ToUpper("hello")                  // "HELLO"
strings.TrimSpace("  hello  ")            // "hello"
strings.Trim("!!!hello!!!", "!")          // "hello"

// String building (efficient concatenation)
var b strings.Builder
b.WriteString("Hello")
b.WriteString(" ")
b.WriteString("World")
result := b.String()  // "Hello World"
```

## strconv (String Conversion)

```go
import "strconv"

// String to int
i, err := strconv.Atoi("123")             // int, error
i64, err := strconv.ParseInt("123", 10, 64)  // base 10, 64-bit

// Int to string
s := strconv.Itoa(123)                    // "123"
s = strconv.FormatInt(int64(123), 10)     // base 10

// String to float
f, err := strconv.ParseFloat("3.14", 64)  // 64-bit

// Float to string
s = strconv.FormatFloat(3.14, 'f', 2, 64) // "3.14", 2 decimals

// String to bool
b, err := strconv.ParseBool("true")       // true, nil

// Bool to string
s = strconv.FormatBool(true)              // "true"
```

## time (Time and Date)

```go
import "time"

// Current time
now := time.Now()

// Create specific time
t := time.Date(2025, time.October, 26, 10, 30, 0, 0, time.UTC)

// Parsing
layout := "2006-01-02 15:04:05"  // Reference time (must use these exact values)
t, err := time.Parse(layout, "2025-10-26 10:30:00")

// Common layouts
time.RFC3339      // "2006-01-02T15:04:05Z07:00"
time.RFC822       // "02 Jan 06 15:04 MST"
"2006-01-02"      // Date only

// Formatting
s := now.Format("2006-01-02 15:04:05")
s = now.Format(time.RFC3339)

// Arithmetic
tomorrow := now.Add(24 * time.Hour)
yesterday := now.Add(-24 * time.Hour)
nextWeek := now.AddDate(0, 0, 7)  // years, months, days

// Duration
duration := 5 * time.Second
time.Sleep(duration)

// Comparison
if t1.Before(t2) { ... }
if t1.After(t2) { ... }
if t1.Equal(t2) { ... }

// Components
year, month, day := now.Date()
hour, min, sec := now.Clock()
weekday := now.Weekday()

// Unix timestamp
timestamp := now.Unix()                    // seconds
timestamp_ms := now.UnixMilli()            // milliseconds
t = time.Unix(timestamp, 0)                // From timestamp
```

## encoding/json (JSON)

```go
import "encoding/json"

type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email,omitempty"`  // Omit if empty
    Password  string    `json:"-"`                 // Never marshal
    CreatedAt time.Time `json:"created_at"`
}

// Marshal (struct to JSON)
user := User{ID: 1, Name: "Alice"}
data, err := json.Marshal(user)
// data = []byte(`{"id":1,"name":"Alice"}`)

// Marshal with indentation (pretty print)
data, err := json.MarshalIndent(user, "", "  ")

// Unmarshal (JSON to struct)
var user User
err := json.Unmarshal(data, &user)

// Decode from reader
decoder := json.NewDecoder(r)
var user User
err := decoder.Decode(&user)

// Encode to writer
encoder := json.NewEncoder(w)
err := encoder.Encode(user)

// Working with unknown structure
var result map[string]interface{}
json.Unmarshal(data, &result)
name := result["name"].(string)
```

## net/http (HTTP)

### HTTP Server

```go
import "net/http"

// Simple handler
func homeHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("Hello, World!"))
}

// Register handlers
http.HandleFunc("/", homeHandler)
http.HandleFunc("/api/users", usersHandler)

// Start server
http.ListenAndServe(":8080", nil)

// With custom server
server := &http.Server{
    Addr:           ":8080",
    Handler:        nil,
    ReadTimeout:    10 * time.Second,
    WriteTimeout:   10 * time.Second,
    MaxHeaderBytes: 1 << 20,
}
server.ListenAndServe()
```

### HTTP Client

```go
// Simple GET
resp, err := http.Get("https://api.example.com/users")
if err != nil {
    return err
}
defer resp.Body.Close()

body, err := io.ReadAll(resp.Body)

// With request
req, err := http.NewRequest("POST", url, bytes.NewBuffer(jsonData))
req.Header.Set("Content-Type", "application/json")

client := &http.Client{Timeout: 10 * time.Second}
resp, err := client.Do(req)

// Check status
if resp.StatusCode != http.StatusOK {
    return fmt.Errorf("unexpected status: %d", resp.StatusCode)
}
```

### Context in HTTP

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // Add timeout
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // Pass to database/API calls
    user, err := db.GetUser(ctx, id)
}
```

## io / io/ioutil

```go
import (
    "io"
    "os"
)

// Copy
io.Copy(dst, src)

// Read all
data, err := io.ReadAll(reader)

// Write string
io.WriteString(w, "Hello")

// Limit reader
limited := io.LimitReader(r, 1024)  // Read max 1KB

// Multi-reader
combined := io.MultiReader(r1, r2, r3)

// Pipe (connect reader and writer)
pr, pw := io.Pipe()
```

## os (Operating System)

### File Operations

```go
import "os"

// Open file for reading
f, err := os.Open("file.txt")
defer f.Close()

// Create file for writing
f, err := os.Create("file.txt")
defer f.Close()

// Open with flags
f, err := os.OpenFile("file.txt", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)

// Read file
data, err := os.ReadFile("file.txt")

// Write file
err := os.WriteFile("file.txt", data, 0644)

// File info
info, err := os.Stat("file.txt")
size := info.Size()
modTime := info.ModTime()
isDir := info.IsDir()

// Check if file exists
if _, err := os.Stat("file.txt"); err == nil {
    // File exists
} else if os.IsNotExist(err) {
    // File does not exist
}

// Remove file
os.Remove("file.txt")

// Remove directory and contents
os.RemoveAll("directory/")
```

### Directory Operations

```go
// Create directory
os.Mkdir("mydir", 0755)

// Create nested directories
os.MkdirAll("path/to/dir", 0755)

// Read directory
entries, err := os.ReadDir(".")
for _, entry := range entries {
    fmt.Println(entry.Name(), entry.IsDir())
}

// Change directory
os.Chdir("/path/to/dir")

// Get working directory
wd, err := os.Getwd()

// Home directory
home, err := os.UserHomeDir()
```

### Environment Variables

```go
// Get env var
value := os.Getenv("DATABASE_URL")

// Set env var
os.Setenv("DATABASE_URL", "postgres://...")

// Get all env vars
for _, env := range os.Environ() {
    fmt.Println(env)
}
```

## path/filepath (File Paths)

```go
import "path/filepath"

// Join paths (OS-specific separator)
path := filepath.Join("home", "user", "file.txt")  // home/user/file.txt

// Split path
dir, file := filepath.Split("/path/to/file.txt")  // "/path/to/", "file.txt"

// Base name
base := filepath.Base("/path/to/file.txt")  // "file.txt"

// Directory name
dir := filepath.Dir("/path/to/file.txt")  // "/path/to"

// Extension
ext := filepath.Ext("file.txt")  // ".txt"

// Absolute path
abs, err := filepath.Abs("relative/path")

// Walk directory tree
filepath.Walk(".", func(path string, info os.FileInfo, err error) error {
    if err != nil {
        return err
    }
    fmt.Println(path)
    return nil
})

// Glob (pattern matching)
matches, err := filepath.Glob("*.go")
```

## bufio (Buffered I/O)

```go
import "bufio"

// Buffered reader
f, _ := os.Open("file.txt")
scanner := bufio.NewScanner(f)

// Read line by line
for scanner.Scan() {
    line := scanner.Text()
    fmt.Println(line)
}

if err := scanner.Err(); err != nil {
    return err
}

// Buffered writer
w := bufio.NewWriter(os.Stdout)
w.WriteString("Hello\n")
w.Flush()  // Important: flush buffer
```

## context (Context)

```go
import "context"

// Background context (root)
ctx := context.Background()

// With cancel
ctx, cancel := context.WithCancel(ctx)
defer cancel()

// With timeout
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

// With deadline
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(ctx, deadline)
defer cancel()

// With value
ctx = context.WithValue(ctx, "user_id", 123)
userID := ctx.Value("user_id").(int)

// Check cancellation
select {
case <-ctx.Done():
    return ctx.Err()  // context.Canceled or context.DeadlineExceeded
default:
    // Continue
}
```

## errors (Error Handling)

```go
import "errors"

// Create error
err := errors.New("something went wrong")

// Wrap error with context
err = fmt.Errorf("failed to get user: %w", err)

// Check error type
if errors.Is(err, sql.ErrNoRows) {
    // Handle specific error
}

// Unwrap error
unwrapped := errors.Unwrap(err)

// Custom error type
type NotFoundError struct {
    Resource string
    ID       int
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s not found: %d", e.Resource, e.ID)
}

// Check custom error
var notFound *NotFoundError
if errors.As(err, &notFound) {
    fmt.Printf("Not found: %s\n", notFound.Resource)
}
```

## sync (Synchronization)

```go
import "sync"

// Mutex
var mu sync.Mutex
mu.Lock()
// Critical section
mu.Unlock()

// RWMutex (multiple readers, single writer)
var mu sync.RWMutex
mu.RLock()
// Read operation
mu.RUnlock()

mu.Lock()
// Write operation
mu.Unlock()

// WaitGroup
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        // Do work
    }(i)
}
wg.Wait()  // Wait for all goroutines

// Once (run exactly once)
var once sync.Once
once.Do(func() {
    // Initialization code
})

// Pool (object reuse)
pool := &sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

buf := pool.Get().(*bytes.Buffer)
defer pool.Put(buf)
```

## regexp (Regular Expressions)

```go
import "regexp"

// Compile regex
re, err := regexp.Compile(`\d+`)

// Must compile (panics on error)
re := regexp.MustCompile(`\d+`)

// Match
matched := re.MatchString("abc123")  // true

// Find
found := re.FindString("abc123def456")  // "123"
all := re.FindAllString("abc123def456", -1)  // ["123", "456"]

// Replace
result := re.ReplaceAllString("abc123", "XXX")  // "abcXXX"

// Submatch (capture groups)
re = regexp.MustCompile(`(\w+)@(\w+\.\w+)`)
matches := re.FindStringSubmatch("user@example.com")
// matches[0] = "user@example.com"
// matches[1] = "user"
// matches[2] = "example.com"
```

## Summary

The Go standard library is comprehensive and production-ready. Prefer stdlib over external libraries when:
- Performance is not critical
- Functionality is covered
- Simplicity is preferred

For complex use cases (ORM, HTTP routing, CLI), use ecosystem libraries (see `@go-ecosystem.md`).
