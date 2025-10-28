---
name: Verifying Actual Functionality
description: Verify the ACTUAL OUTCOME happens, not just that tests pass or commands don't error
when_to_use: when implementing features, fixing bugs, or reporting completion status - before claiming anything works
version: 1.0.0
languages: all
---

# Verifying Actual Functionality

## Overview

**Core principle:** "Tests pass" ≠ "functionality works". Always verify the actual intended outcome happens.

**This skill prevents:** Claiming features work when they only pretend to work (commands succeed but nothing actually happens).

## The Iron Law

```
NO "WORKS" OR "DONE" CLAIM WITHOUT VERIFYING THE ACTUAL OUTCOME
```

**Not sufficient:**

- ❌ Unit tests pass
- ❌ E2E tests pass
- ❌ Command returns success
- ❌ No errors in logs
- ❌ Linter clean

**Required:**

- ✅ Verify the ACTUAL THING the user wants actually happened

## When to Use

Use this BEFORE claiming:

- "Feature is complete"
- "Bug is fixed"
- "Production ready"
- "Tests verify X works"
- "Implementation done"

**Symptoms you're about to violate:**

- Feeling confident because tests pass
- Reporting success based on "no errors"
- Assuming test names describe what tests actually verify
- Skipping verification because "it should work"

## The Verification Checklist

For EVERY feature/fix, answer:

### 1. What is the actual intended outcome?

**Not:** "Deploy command succeeds"
**Yes:** "Container is running on the server with correct image"

**Not:** "Backup command returns success"
**Yes:** "Backup file exists and contains the data"

**Not:** "Tests pass"
**Yes:** "User can do [specific action] and [specific result] happens"

### 2. How do I verify that outcome actually happened?

**Examples:**

**Web service deployment:**

```bash
# After "shipyard deploy production"
ssh server "docker ps | grep myapp"  # Container running?
ssh server "docker logs myapp"        # Logs show startup?
curl https://myapp.com                # Actually responds?
```

**Database backup:**

```bash
# After "shipyard backup create production db data"
ssh server "ls -la /backups/"                    # File exists?
ssh server "tar -tzf /backups/backup-123.tar.gz" # Contains files?
```

**Blue-green deployment:**

```bash
# After deployment
curl https://myapp.com                # New version responding?
ssh server "docker ps | grep blue"    # Blue container running?
ssh server "docker ps | grep green"   # Green container stopped?
```

### 3. What should I see vs. what am I seeing?

**Document expected outcome:**

```
Expected: docker ps shows myapp-web container with image myapp:latest
Actual: [run the command and see what happens]
```

If Expected ≠ Actual, it doesn't work. No matter what tests say.

## Common Violations and Reality

| Excuse                                   | Reality                                                             |
| ---------------------------------------- | ------------------------------------------------------------------- |
| "Unit tests pass"                        | Unit tests test units, not integration or actual outcomes           |
| "E2E tests pass"                         | E2E tests might only check commands don't error, not actual results |
| "No errors in output"                    | Success messages can be fake (lying)                                |
| "Test is named 'TestDeployment'"         | Test name ≠ what test actually verifies                             |
| "I read the code, it should work"        | Should ≠ does. Verify.                                              |
| "Infrastructure works, so feature works" | Networking working ≠ deployment logic exists                        |
| "It's just a small change"               | Small changes break things. Verify.                                 |
| "I'm confident it works"                 | Confidence ≠ verification. Verify.                                  |

**All of these mean: Go verify the actual outcome. No exceptions.**

## Red Flags - STOP

If you're about to say/think:

- "Tests pass, so it works"
- "Command succeeded, so it's done"
- "I fixed the infrastructure, so the feature works"
- "The test name says it tests X, so X must work"
- "I don't need to verify, I'm confident"
- "Verification would take too long"
- "I'll verify if user reports issues"

**STOP. Go verify the actual outcome right now.**

## TODO Handling

**When you see a TODO in critical code path:**

```go
// TODO: Build Docker image locally if needed
```

**This means:**

- ❌ NOT "minor detail to add later"
- ✅ **CORE FUNCTIONALITY IS MISSING**

**Action:**

1. Stop claiming the feature works
2. Document: "TODO at line X means [functionality] is not implemented"
3. Either implement it, or report: "Feature incomplete - TODO at [location]"

**Never dismiss TODOs as "nice-to-haves" if they're in the critical path.**

## Test Analysis

**When analyzing tests, check what they ACTUALLY verify:**

```typescript
test("deploys web service", async () => {
  const result = await cli.run("deploy", "production");
  expect(result.exitCode).toBe(0); // Only checks: command doesn't error
  expect(result.output).toContain("✓ Deployment complete"); // Only checks: output contains success message
});
```

**This test verifies:**

- Command doesn't error ✅
- Output contains success message ✅

**This test DOES NOT verify:**

- Container is deployed ❌
- Container is running ❌
- Service is accessible ❌

**Don't assume test name = what test verifies. Read the test code.**

## Reporting Completion

**Before claiming "done" or "works":**

1. State what you verified:

```
Verified:
- Ran: docker ps on server
- Saw: myapp-web container running with image myapp:latest
- Ran: curl https://myapp.com
- Saw: HTTP 200 with expected content
```

2. State what you didn't verify:

```
Not verified:
- Blue-green switching (would need multiple deployments)
- Rollback functionality (would need intentional failure)
```

3. Never claim unverified functionality works

**Template:**

```
Status: [Feature/Fix] is complete

Verified:
- [Actual action taken]: [Actual result observed]
- [Actual action taken]: [Actual result observed]

Not verified:
- [What wasn't tested]: [Why]

Known issues:
- [Any discovered problems]
```

## Implementation Workflow

1. **Understand what should actually happen**
   - "What does the user expect to see/get/do?"

2. **Implement the feature**
   - Write the code

3. **Verify it actually works**
   - Run the actual commands
   - Check the actual results
   - Compare expected vs actual

4. **Report what you verified**
   - Be specific about what you tested
   - Be honest about what you didn't test

**Do NOT:**

- Skip step 3
- Assume tests verify actual outcomes
- Report "works" based on "tests pass"

## Examples

### Good: Actual Verification

```
Status: Deployment feature complete

Verified:
- Ran: shipyard deploy production
- Checked: docker ps shows myapp-web container
- Verified: Container has correct image tag
- Tested: curl to service returns expected response

Not verified:
- Multiple concurrent deployments
- Deployment with very large images (>5GB)

Known issue:
- Health checks timeout after 30s even if service needs 60s to start
```

### Bad: No Verification

```
Status: Deployment feature complete

Verified:
- All unit tests pass
- E2E tests pass
- Linter clean
- No errors during deployment

Conclusion: Production ready
```

**Why bad:** Never checked if actual container deploys. "Tests pass" is not verification.

## The Bottom Line

**"Tests pass" is not the same as "works".**

Before claiming anything is done:

1. Know what the actual outcome should be
2. Verify that outcome actually happens
3. Report specifically what you verified

If you can't verify because you don't have access/environment, say:
"Cannot verify - would need [environment/access]. Tests pass but actual functionality not confirmed."

**Never claim functionality works without verifying the actual outcome.**
