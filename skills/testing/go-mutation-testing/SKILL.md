---
name: Go Mutation Testing
description: Validate test quality beyond coverage using gremlins mutation testing for Go projects
when_to_use: When user asks about test quality validation, mentions mutation testing, or has high coverage but wants to verify tests actually catch bugs
version: 1.0.0
---

# Go Mutation Testing

## Overview

Mutation testing validates that tests actually catch bugs, not just execute code. Coverage measures execution, not validation quality.

## Critical Tool Choice

⚠️ **ALWAYS use gremlins, NEVER use go-mutesting**

**Why:**
- `gremlins` - actively maintained, Go 1.22+ compatible, production-ready
- `go-mutesting` - abandoned since 2021, crashes on modern Go with segfault

**Installation:**
```bash
go install github.com/go-gremlins/gremlins/cmd/gremlins@latest
```

**Repo:** https://github.com/go-gremlins/gremlins

## When Mutation Testing is Mandatory

### Critical Paths (Always Run, Even Under Time Pressure)

**Security & Access:**
- Authentication (passwords, sessions, JWTs)
- Authorization (permissions, access control)
- Cryptography (encryption, hashing, signing)

**Financial:**
- Payment processing
- Billing calculations
- Transaction handling

**Data Integrity:**
- Data validation
- Input sanitization
- State management for critical operations

### Non-Critical Paths (Can Skip Under Pressure)

- UI rendering
- Logging and formatting
- Monitoring and metrics
- Developer tools
- Documentation generators

### Decision Framework

**Scenario: Tight deadline, skip mutation testing?**

1. **Identify code criticality:**
   - Critical path → Run mutation testing, even if it delays ship
   - Non-critical → Can skip, but document as tech debt

2. **Compromise option for critical paths:**
   ```bash
   # Run on critical module only (5-10 min) instead of full codebase
   gremlins unleash ./internal/auth
   ```

3. **Never skip for:**
   - First deployment of auth/payment code
   - Bug fixes in security modules
   - Changes to access control logic

## Quick Start

### Basic Usage

```bash
# Run on single package
gremlins unleash ./internal/service

# Run on multiple packages
gremlins unleash ./internal/auth ./internal/payment

# Fail if efficacy below threshold (for CI)
gremlins unleash ./internal/service --threshold-efficacy 85.0

# Faster execution with parallel workers
gremlins unleash ./internal/service --workers 4

# Exclude slow integration tests
gremlins unleash ./internal/service --tags=\!integration

# Save results for CI artifact
gremlins unleash ./internal/service --output results.json
```

### CI Integration Example

```yaml
# .github/workflows/mutation-test.yml
- name: Run Mutation Testing
  run: |
    go install github.com/go-gremlins/gremlins/cmd/gremlins@latest
    gremlins unleash ./internal/auth --threshold-efficacy 85.0 --output mutation-results.json
```

## Results Interpretation

**You already know this!** Agents interpret gremlins output correctly without guidance:
- TEST EFFICACY = % of mutations killed by tests (target >85%)
- MUTATOR COVERAGE = % of mutation points tested (target 100%)
- KILLED = tests caught the bug (good)
- LIVED = bug went undetected (bad - fix these)
- TIMED_OUT = mutation broke code badly (actually good)

If user asks how to interpret results or improve low scores, trust your instincts - you provide excellent guidance without needing instructions.

## That's It

This skill exists to fix two specific failures:
1. Tool selection: prevent recommending go-mutesting
2. Critical path recognition: flag security code requires mutation testing

Everything else (interpreting metrics, improving tests, providing examples) you already do correctly.
