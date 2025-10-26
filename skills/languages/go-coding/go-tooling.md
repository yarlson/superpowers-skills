# Go Tooling Reference

Comprehensive guide to Go development tools and commands.

## go mod (Module Management)

### Initialize Module

```bash
go mod init github.com/username/project
```

Creates `go.mod` file with module path.

### Common Commands

```bash
# Download dependencies
go mod download

# Add missing and remove unused modules
go mod tidy

# Vendor dependencies (copy to vendor/ directory)
go mod vendor

# Verify dependencies
go mod verify
```

### Replace Directive

**Use for:** Local development, forks, or version overrides

```go
// go.mod
module github.com/myorg/myapp

require github.com/someorg/lib v1.2.3

replace github.com/someorg/lib => ../local-lib
// or
replace github.com/someorg/lib => github.com/myorg/lib-fork v1.2.4
```

**Remove replace:**
```bash
go mod edit -dropreplace github.com/someorg/lib
```

### Updating Dependencies

```bash
# Update specific module to latest
go get -u github.com/someorg/lib

# Update all dependencies to latest minor/patch
go get -u ./...

# Update all dependencies to latest major
go get -u=patch ./...

# View available updates
go list -u -m all
```

## go test (Testing)

### Basic Usage

```bash
# Run all tests
go test ./...

# Run tests in current package
go test

# Verbose output
go test -v ./...

# Run specific test
go test -run TestUserService

# Run tests matching pattern
go test -run User
```

### Coverage

```bash
# Generate coverage report
go test -cover ./...

# Detailed coverage by function
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out

# HTML coverage report
go tool cover -html=coverage.out
```

**Target:** Aim for >80% coverage on business logic.

### Race Detector

**Always run before deploying concurrent code:**

```bash
# Test with race detector
go test -race ./...

# Run app with race detector
go run -race main.go
```

**What it catches:**
- Concurrent reads/writes to same variable
- Data races in goroutines
- Unsafe concurrent map access

**Cost:** Slower execution (~10x), higher memory

### Benchmarks

```go
func BenchmarkUserService_GetUser(b *testing.B) {
    service := setupService()
    for i := 0; i < b.N; i++ {
        service.GetUser(1)
    }
}
```

```bash
# Run benchmarks
go test -bench=. ./...

# With memory allocation stats
go test -bench=. -benchmem ./...

# Run specific benchmark
go test -bench=GetUser
```

### Test Flags

```bash
# Timeout (default 10m)
go test -timeout 30s ./...

# Short mode (skip long tests)
go test -short ./...

# Parallel tests
go test -parallel 4 ./...

# Fail fast (stop on first failure)
go test -failfast ./...

# Run tests with tags
go test -tags=integration ./...
```

**Build tags in test files:**
```go
//go:build integration

package mypackage

func TestIntegration(t *testing.T) {
    // Only runs with -tags=integration
}
```

### Table-Driven Test Pattern

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -1, -2},
        {"zero", 0, 5, 5},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

## go fmt / goimports (Formatting)

### go fmt

**Auto-formats Go code to standard style:**

```bash
# Format current directory
go fmt ./...

# Format specific file
go fmt main.go
```

**Editor integration:** Set up format-on-save in your editor. Never manually format code.

### goimports

**Better than go fmt - also manages imports:**

```bash
# Install
go install golang.org/x/tools/cmd/goimports@latest

# Format with import management
goimports -w .
```

**Automatically:**
- Adds missing imports
- Removes unused imports
- Groups imports (stdlib → external → internal)
- Formats code

**Editor setup:** Use `goimports` instead of `gofmt` for format-on-save.

## golangci-lint (Linting)

**Meta-linter running multiple linters:**

### Installation

```bash
# macOS
brew install golangci-lint

# Linux
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

# Or via go install
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

### Basic Usage

```bash
# Run with default linters
golangci-lint run

# Run with all linters
golangci-lint run --enable-all

# Auto-fix issues
golangci-lint run --fix

# Run specific linters
golangci-lint run --enable=errcheck,gosimple,staticcheck
```

### Configuration

Create `.golangci.yml` in project root:

```yaml
linters:
  enable:
    - errcheck      # Check error returns
    - gosimple      # Simplify code
    - govet         # Go vet
    - ineffassign   # Detect ineffective assignments
    - staticcheck   # Static analysis
    - unused        # Detect unused code
    - misspell      # Spelling
    - gofmt         # Format check
    - goimports     # Import check

linters-settings:
  errcheck:
    check-blank: true  # Check _ assignments

  govet:
    check-shadowing: true

issues:
  exclude-use-default: false
  max-issues-per-linter: 0
  max-same-issues: 0

run:
  timeout: 5m
  tests: true
```

### Recommended Linters

| Linter | Purpose |
|--------|---------|
| errcheck | Checks unchecked errors |
| gosimple | Simplifies code |
| govet | Official Go analysis |
| staticcheck | Advanced static analysis |
| unused | Finds unused code |
| ineffassign | Detects useless assignments |
| misspell | Catches spelling errors |
| gofmt | Ensures formatting |
| goimports | Checks import organization |

### CI Integration

```yaml
# .github/workflows/lint.yml
- name: golangci-lint
  uses: golangci/golangci-lint-action@v3
  with:
    version: latest
```

## go build (Building)

### Basic Build

```bash
# Build current package
go build

# Build specific file
go build main.go

# Build and specify output
go build -o myapp

# Build for all packages
go build ./...
```

### Cross-Compilation

```bash
# Linux
GOOS=linux GOARCH=amd64 go build -o myapp-linux

# Windows
GOOS=windows GOARCH=amd64 go build -o myapp.exe

# macOS
GOOS=darwin GOARCH=amd64 go build -o myapp-mac

# macOS ARM (M1/M2)
GOOS=darwin GOARCH=arm64 go build -o myapp-mac-arm
```

**Common platforms:**
- `linux/amd64` - Linux servers
- `darwin/amd64` - Intel Macs
- `darwin/arm64` - Apple Silicon Macs
- `windows/amd64` - Windows

### Build Tags

```go
//go:build prod

package config

const Environment = "production"
```

```bash
# Build with tag
go build -tags=prod

# Multiple tags
go build -tags="prod,feature_x"
```

### Version Injection

**Inject version at build time:**

```go
package main

var (
    version = "dev"
    commit  = "unknown"
    date    = "unknown"
)

func main() {
    fmt.Printf("Version: %s\nCommit: %s\nDate: %s\n", version, commit, date)
}
```

```bash
# Build with version info
go build -ldflags "\
  -X main.version=1.2.3 \
  -X main.commit=$(git rev-parse HEAD) \
  -X main.date=$(date -u +%Y-%m-%d)"
```

### Optimization Flags

```bash
# Disable debug info (smaller binary)
go build -ldflags="-s -w"

# Disable inlining (faster builds)
go build -gcflags="-N -l"

# Optimize for size
go build -ldflags="-s -w" -trimpath
```

## go generate (Code Generation)

**Run code generators before build:**

```go
//go:generate moq -out mock_user_repo.go . UserRepository
//go:generate stringer -type=Status
```

```bash
# Run all generators
go generate ./...

# Verbose output
go generate -v ./...
```

**Common uses:**
- Mock generation (moq, mockgen)
- String generation (stringer)
- Asset embedding
- Protocol buffers

## Debugging

### Delve

**Go debugger:**

```bash
# Install
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug current package
dlv debug

# Debug with arguments
dlv debug -- --config=dev.yaml

# Debug tests
dlv test

# Attach to running process
dlv attach <pid>
```

**Basic commands:**
```
break main.main    # Set breakpoint
continue           # Continue execution
next               # Step over
step               # Step into
print myVar        # Print variable
locals             # Show local variables
exit               # Quit
```

### pprof (Profiling)

**CPU profiling:**

```go
import (
    "runtime/pprof"
    "os"
)

f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
```

```bash
# Analyze profile
go tool pprof cpu.prof

# Web UI
go tool pprof -http=:8080 cpu.prof
```

**Memory profiling:**

```go
import "runtime/pprof"

f, _ := os.Create("mem.prof")
pprof.WriteHeapProfile(f)
f.Close()
```

**HTTP profiling (live server):**

```go
import _ "net/http/pprof"

go func() {
    http.ListenAndServe(":6060", nil)
}()
```

```bash
# View profiles
go tool pprof http://localhost:6060/debug/pprof/profile
go tool pprof http://localhost:6060/debug/pprof/heap
```

## go vet (Static Analysis)

```bash
# Run go vet
go vet ./...

# Specific checks
go vet -composites=false ./...
```

**What it catches:**
- Printf format errors
- Unreachable code
- Invalid struct tags
- Shadowed variables
- Suspicious constructs

## go doc (Documentation)

```bash
# View package documentation
go doc fmt

# View function documentation
go doc fmt.Printf

# View all documentation
go doc -all fmt
```

## Development Workflow

### Typical Commands

```bash
# During development
goimports -w .                  # Format + organize imports
go test -race ./...             # Run tests with race detector
golangci-lint run --fix         # Lint and auto-fix

# Before commit
go mod tidy                     # Clean dependencies
go test -cover ./...            # Check coverage
golangci-lint run               # Final lint check

# Before release
go test -race -cover ./...      # Full test suite
go build -ldflags="-s -w"       # Optimized build
```

### Makefile Example

```makefile
.PHONY: test build lint fmt

fmt:
	goimports -w .

lint:
	golangci-lint run

test:
	go test -race -cover ./...

build:
	go build -o bin/myapp

clean:
	rm -rf bin/

all: fmt lint test build
```

## CI/CD Example

```yaml
# .github/workflows/go.yml
name: Go

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-go@v4
        with:
          go-version: '1.21'

      - name: Install dependencies
        run: go mod download

      - name: Format check
        run: |
          go install golang.org/x/tools/cmd/goimports@latest
          test -z $(goimports -l .)

      - name: Lint
        uses: golangci/golangci-lint-action@v3

      - name: Test
        run: go test -race -cover ./...

      - name: Build
        run: go build -v ./...
```

## Summary

| Tool | Command | Purpose |
|------|---------|---------|
| **go mod** | `go mod tidy` | Manage dependencies |
| **go test** | `go test -race ./...` | Run tests with race detection |
| **goimports** | `goimports -w .` | Format and organize imports |
| **golangci-lint** | `golangci-lint run` | Comprehensive linting |
| **go build** | `go build -o app` | Build binary |
| **go generate** | `go generate ./...` | Run code generators |
| **dlv** | `dlv debug` | Interactive debugging |
| **pprof** | `go tool pprof` | Performance profiling |

**Bottom line:** Set up these tools early in your project. Automate them in CI/CD. Run them before every commit.
