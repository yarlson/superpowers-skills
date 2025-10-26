---
name: HTMX Attributes with plainkit/htmx
description: Complete plainkit/htmx attribute reference - HTTP methods, targeting, swaps, triggers, synchronization, OOB updates, file uploads, progressive enhancement
when_to_use: when building Go web apps with plainkit/htmx and need HTMX attributes for dynamic interactions (infinite scroll, inline edit, live validation, modal dialogs, file uploads, multi-element updates). For core HTML see skills/plainkit/html, for architecture see skills/plainkit/webapp.
version: 1.0.0
---

# HTMX Attributes with plainkit/htmx

## Overview

plainkit/htmx provides type-safe Go functions for all HTMX attributes, enabling hypermedia-driven interactions without JavaScript. Each function returns `html.Global`, making HTMX attributes work with any HTML element.

**Core principle:** Build interactive web applications using declarative HTML attributes and server-side rendering, avoiding client-side JavaScript complexity.

**Related skills:**

- skills/plainkit/html - Core HTML generation and type system
- skills/plainkit/webapp - Architecture patterns and workflows

## ⚠️ CRITICAL: HTTP Status Codes and Error Handling

**HTMX ONLY SWAPS CONTENT ON 2xx STATUS CODES BY DEFAULT**

This is the #1 mistake developers make with HTMX:

### The Problem

```go
// ❌ WRONG - HTMX will NOT swap this response
func HandleError(w http.ResponseWriter, r *http.Request) {
    errorHTML := RenderErrorForm("Invalid input")
    w.WriteHeader(http.StatusBadRequest) // 400 - HTMX ignores this!
    w.Write([]byte(errorHTML))
}
```

**What happens:** HTMX receives the response but **does not perform the swap** because the status code is 400. The error HTML is sent but never displayed. The user sees nothing.

### The Solution

**For HTMX requests, return 200 OK and convey errors through HTML content:**

```go
// ✅ CORRECT - HTMX will swap this response
func HandleError(w http.ResponseWriter, r *http.Request) {
    isHTMX := r.Header.Get("HX-Request") == "true"

    if isHTMX {
        // Return 200 with error in HTML content
        errorHTML := RenderErrorForm("Invalid input") // Form with error banner
        w.WriteHeader(http.StatusOK) // 200 - HTMX will swap
        w.Write([]byte(errorHTML))
    } else {
        // Non-HTMX: use proper HTTP semantics
        w.WriteHeader(http.StatusBadRequest) // 400 for full page
        w.Write([]byte(RenderFullErrorPage("Invalid input")))
    }
}
```

### Key Rules

1. **HTMX validation errors:** Return `200 OK` with error displayed in HTML (error banner, red text, etc.)
2. **Non-HTMX requests:** Use proper HTTP status codes (400, 404, 500, etc.)
3. **Detection:** Check `HX-Request` header to differentiate
4. **Error display:** Put errors in the HTML content (e.g., form with error message)

### Why This Design?

HTMX is designed for **hypermedia exchanges**, not JSON APIs. Errors are conveyed through the HTML response (error messages, validation feedback), not HTTP status codes. This allows:

- Progressive enhancement (graceful degradation without JS)
- Semantic HTML error display
- Consistent UX patterns

### Alternative: Response-Targets Extension

If you MUST use 4xx status codes, use the `response-targets` extension:

```go
// In HTML: add extension
Div(
    HxExt("response-targets"),
    HxPost("/submit"),
    HxTarget("#success-target"),           // 2xx responses
    HxTargetError("#error-target"),        // 4xx/5xx responses
)
```

But this is more complex and not the recommended approach for most cases.

### Summary

| Scenario              | Status Code | Action                              |
| --------------------- | ----------- | ----------------------------------- |
| HTMX success          | 200 OK      | Swap content                        |
| HTMX validation error | 200 OK      | Swap content (with error displayed) |
| HTMX server error     | 500         | No swap (or use response-targets)   |
| Non-HTMX error        | 400/500     | Full page render with proper status |

**Remember:** For HTMX, semantic meaning is in the HTML, not the status code.

## Quick Reference

### HTTP Methods

```go
HxGet(url)      // GET request
HxPost(url)     // POST request
HxPut(url)      // PUT request
HxDelete(url)   // DELETE request
HxPatch(url)    // PATCH request
```

### Targeting & Swapping

```go
HxTarget(selector)  // Where to place response
HxSwap(strategy)    // How to swap content
HxSwapOob(value)    // Out-of-band multi-element updates
```

### Triggers & Events

```go
HxTrigger(event)           // What triggers request
HxIndicator(selector)      // Loading indicator
HxDisabledElt(selector)    // Disable during request
```

### Request Synchronization

```go
HxSync(strategy)     // Prevent race conditions
HxInclude(selector)  // Include form fields
HxParams(filter)     // Filter parameters
```

### Navigation

```go
HxPushUrl(url)     // Add history entry
HxReplaceUrl(url)  // Replace history entry
HxBoost(bool)      // Progressive enhancement
```

## Complete Swap Strategies

HTMX provides 9 swap strategies for different content update patterns:

| Strategy      | Effect                              | Common Use Case                  |
| ------------- | ----------------------------------- | -------------------------------- |
| `innerHTML`   | Replace inner HTML (default)        | Update content area              |
| `outerHTML`   | Replace entire element              | Swap display/edit modes          |
| `textContent` | Replace text only (no HTML parsing) | Safe user text display           |
| `beforebegin` | Insert before element               | Prepend siblings                 |
| `afterbegin`  | Insert before first child           | Prepend children                 |
| `beforeend`   | Insert after last child             | Infinite scroll, append items    |
| `afterend`    | Insert after element                | Append siblings                  |
| `delete`      | Delete element                      | Remove items (ignore response)   |
| `none`        | No swap (OOB only)                  | Side effects without main update |

### Swap Modifiers

Enhance swap behavior with modifiers:

```go
HxSwap("innerHTML swap:200ms")           // 200ms transition
HxSwap("innerHTML settle:500ms")         // 500ms settle time
HxSwap("beforeend scroll:bottom")        // Scroll to bottom after
HxSwap("innerHTML show:top")             // Show top of swapped content
HxSwap("innerHTML focus-scroll:true")    // Focus scrolls into view
```

**Combine modifiers:** `"innerHTML swap:100ms settle:200ms scroll:bottom"`

## Request Synchronization

Prevent race conditions and double submissions with `HxSync`:

```go
// Drop new requests while one is in-flight
Button(
    HxPost("/submit"),
    HxSync("this:drop"),  // Ignore clicks during request
    T("Submit"),
)

// Abort current request, start new one
Input(
    HxGet("/search"),
    HxTrigger("keyup changed delay:500ms"),
    HxSync("this:replace"),  // Latest search wins
)

// Queue requests (execute in order)
Button(
    HxPost("/process"),
    HxSync("this:queue first"),  // Queue, process first request
)
```

**Sync Strategies:**

- `drop` - Ignore new request if one in-flight
- `abort` - Cancel current, run new one
- `replace` - Cancel current, replace with new
- `queue [first|last|all]` - Queue requests

**Syntax:** `HxSync("selector:strategy")` - Use CSS selector to target synchronization scope

## Out-of-Band (OOB) Swaps

Update multiple page elements from a single response:

```go
// Server response updates both main target AND cart counter
func (h *Handler) AddToCart(w http.ResponseWriter, r *http.Request) {
    // Add item logic...

    fragment := Fragment(
        // Main response (goes to hx-target)
        Div(T("Item added!")),

        // Out-of-band swap - updates cart counter by ID
        Span(
            AId("cart-count"),
            HxSwapOob("true"),  // or "innerHTML", "outerHTML"
            T(fmt.Sprintf("%d items", cartCount)),
        ),
    )

    w.Write([]byte(Render(fragment)))
}

// Trigger button
Button(
    HxPost("/cart/add"),
    HxTarget("#message"),  // Main target
    T("Add to Cart"),
)
```

**OOB Swap Values:**

- `"true"` - outerHTML (replace entire element)
- `"innerHTML"` - Replace content only
- `"outerHTML:#custom-id"` - Explicit target
- `"beforeend:#list"` - Append to element

**Important:** OOB elements MUST have `id` attribute to identify swap target

## Trigger Syntax

Control when requests are sent with flexible trigger syntax:

```go
// Basic triggers
HxTrigger("click")           // On click (default for buttons)
HxTrigger("submit")          // On form submit
HxTrigger("change")          // On input change
HxTrigger("keyup")           // On key release

// Modifiers
HxTrigger("keyup changed delay:500ms")     // Debounce, only if value changed
HxTrigger("click once")                    // Fire only once
HxTrigger("click throttle:1s")             // Throttle to 1s intervals
HxTrigger("mouseenter once delay:2s")      // Wait 2s on first hover

// Special triggers
HxTrigger("load")           // On element load
HxTrigger("revealed")       // On element scrolled into view
HxTrigger("intersect")      // On intersection observer trigger

// Multiple triggers
HxTrigger("change, keyup delay:500ms")     // OR condition

// Event filters
HxTrigger("click[ctrlKey]")                // Only if Ctrl held
HxTrigger("keyup[key=='Enter']")           // Only on Enter key
```

**Common Patterns:**

- Search: `"keyup changed delay:500ms"`
- Infinite scroll: `"revealed"`
- Auto-save: `"change delay:1s"`
- Click-once buttons: `"click once"`

## Form Composition

Include values from elements outside the current element:

```go
// Include specific input by selector
Button(
    HxPost("/submit"),
    HxInclude("[name='email']"),  // Include email input
    T("Submit"),
)

// Include multiple elements
Input(
    HxGet("/validate/username"),
    HxInclude("#email, #phone"),  // Cross-field validation
)

// Include entire container
HxInclude("#form-container")  // All inputs within

// Filter which params to send
Input(
    HxGet("/search"),
    HxParams("query, limit"),  // Only these params
)
HxParams("*")       // All params (default)
HxParams("none")    // No params
```

## File Uploads

Handle multipart form data for file uploads:

```go
Form(
    HxPost("/upload"),
    HxEncoding("multipart/form-data"),  // Required for files
    HxTarget("#result"),

    Input(AType("file"), AName("avatar")),
    Button(AType("submit"), T("Upload")),

    // Optional: Progress indicator
    Div(
        AId("progress"),
        AClass("htmx-indicator"),
        T("Uploading..."),
    ),
)
```

**Key Requirements:**

- `HxEncoding("multipart/form-data")` is mandatory
- Can be on `<form>` or `<button>`
- Server handles as standard multipart upload

## History Management

Control browser URL and history:

```go
// Push URL (adds history entry, enables back button)
Button(
    HxGet("/page/2"),
    HxPushUrl("true"),      // Push request URL
    HxTarget("#content"),
    T("Next Page"),
)

// Replace URL (no history entry)
Button(
    HxGet("/filter?type=active"),
    HxReplaceUrl("true"),   // Update URL, no back button entry
    HxTarget("#list"),
    T("Show Active"),
)

// Custom URL (different from request URL)
HxPushUrl("/custom-url")
HxReplaceUrl("/different-url")
```

**When to Use:**

- **Push:** Navigation where user expects back button (tabs, pages, filters)
- **Replace:** Temporary states where back button shouldn't reverse every action (form steps, modals)

**Important:** If you push a URL, that URL MUST return a full page (for bookmarks/refreshes)

## Progressive Enhancement

Boost regular links and forms with AJAX:

```go
// Enable boost for entire section
Body(
    HxBoost(true),

    A(AHref("/page"), T("Link")),      // Becomes AJAX GET
    Form(                               // Becomes AJAX submit
        AAction("/submit"),
        AMethod("POST"),
        Input(AType("text"), AName("query")),
        Button(AType("submit"), T("Search")),
    ),
)
```

**Boost Behavior:**

- Converts links to AJAX GET
- Converts forms to AJAX using form method
- Targets `<body>` with `innerHTML` by default
- Only same-domain, non-anchor links boosted
- Updates browser history automatically

**Use Cases:**

- Progressive enhancement (works without JS)
- Quick HTMX adoption without HTML changes
- Faster navigation for multi-page apps

## User Interaction

Add confirmation and prompts:

```go
// Confirmation dialog
Button(
    HxDelete("/users/123"),
    HxConfirm("Are you sure you want to delete this user?"),
    T("Delete"),
)

// Prompt dialog (input value sent with request)
Button(
    HxPost("/users/rename"),
    HxPrompt("Enter new name:"),
    T("Rename"),
)
```

## Advanced Attributes

### Extensions

```go
// Enable HTMX extensions
Div(
    HxExt("json-enc"),           // JSON encoding
    HxPost("/api/data"),
)

// Multiple extensions
HxExt("json-enc, response-targets")
```

**Common Extensions:**

- `json-enc` - Send JSON instead of form-data
- `morphdom` - DOM morphing for minimal updates
- `response-targets` - Custom targets based on response status

### Content Selection

```go
// Select subset of response
HxGet("/page"),
HxSelect("#main-content")       // Only extract this element

// Out-of-band selection
HxSelectOob("#notification")    // OOB swap from this element
```

### Disinheritance

```go
// Prevent child elements from inheriting attributes
Div(
    HxGet("/data"),
    HxTarget("#result"),

    Button(
        HxDisinherit("*"),      // Don't inherit any hx-* attributes
        HxPost("/other"),       // Use own attributes only
    ),
)
```

### Preservation

```go
// Preserve elements during swaps (iframes, videos, etc.)
Iframe(
    ASrc("/video"),
    HxPreserve(),  // Don't swap this element
)
```

### Real-Time Features

```go
// Server-sent events
Div(
    HxSse("connect:/events"),
    HxTrigger("sse:message"),
)

// WebSockets
Div(
    HxWs("connect:/ws"),
)
```

## Unified Endpoints: Full Page + Fragment Pattern

**Best Practice:** Use a single endpoint that serves both full pages and HTMX fragments based on the `HX-Request` header. This eliminates duplicate routes and ensures consistent filtering/state management.

### The Pattern

```go
// Single endpoint: GET /items
func (h *Handler) HandleItems(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // Parse query parameters (filters, pagination, etc.)
    filter := r.URL.Query().Get("status")

    // Business logic (same for both cases)
    items, err := h.store.GetItems(ctx, filter)
    if err != nil {
        http.Error(w, "Failed to load items", http.StatusInternalServerError)
        return
    }

    // Check if this is an HTMX request
    if r.Header.Get("HX-Request") == "true" {
        // Return fragment only (for polling/partial updates)
        w.Header().Set("Content-Type", "text/html; charset=utf-8")
        _, _ = fmt.Fprint(w, html.Render(components.ItemList(ctx, items)))
        return
    }

    // Build full request URL with query params for polling
    requestURL := r.URL.Path
    if r.URL.RawQuery != "" {
        requestURL = r.URL.Path + "?" + r.URL.RawQuery
    }

    // Return full page (for direct access/refresh)
    w.Header().Set("Content-Type", "text/html; charset=utf-8")
    page := pages.ItemsPage(ctx, items, requestURL)
    _, _ = fmt.Fprint(w, "<!DOCTYPE html>\n")
    _, _ = fmt.Fprint(w, html.Render(page))
}
```

### Template: Pass Request URL for Polling

```go
// pages/items.go
func ItemsPage(ctx context.Context, items []*Item, requestURL string) Node {
    return layout.Base(
        "Items",
        Div(
            H1(T("Items")),

            // Filter controls (static - outside polling container)
            Div(
                A(AHref("/items?status=active"), T("Active")),
                A(AHref("/items?status=completed"), T("Completed")),
            ),

            // Polling container - uses SAME URL (preserves filters!)
            Div(
                AId("items-container"),
                HxGet(requestURL),  // ✅ Includes query params like ?status=active
                HxTrigger("every 5s"),
                HxSwap("innerHTML"),

                // Fragment content
                components.ItemList(ctx, items),
            ),
        ),
    )
}
```

### Benefits

1. **Single source of truth** - One endpoint, one set of business logic
2. **Automatic filter preservation** - Polling inherits all query parameters
3. **Simpler routing** - No separate `/items/table` endpoint needed
4. **Consistent behavior** - Both full page and fragments see same data/filters
5. **Easier testing** - Test one endpoint, both modes work

### When to Use This Pattern

- ✅ Polling/auto-refresh with filters
- ✅ Pages with query parameters (search, pagination, filters)
- ✅ Any endpoint that needs both full page and fragment views
- ❌ Complex forms with different validation rules for HTMX vs non-HTMX
- ❌ Endpoints that fundamentally return different data structures

## Common Patterns

### Infinite Scroll

```go
Div(
    AId("item-list"),
    Fragment(items...),

    Button(
        AId("load-more"),
        HxGet("/items/more?offset=20"),
        HxTarget("#item-list"),
        HxSwap("beforeend"),        // Append items
        HxTrigger("revealed"),      // Auto-trigger on scroll into view
        T("Load More"),
    ),
)
```

### Inline Edit

```go
// Display mode
Div(
    AId("user-123"),
    Span(T(user.Name)),
    Button(
        HxGet("/users/123/edit"),
        HxTarget("#user-123"),
        HxSwap("outerHTML"),    // Replace entire element
        T("Edit"),
    ),
)

// Edit mode (server returns this)
Form(
    AId("user-123"),
    HxPost("/users/123"),
    HxTarget("#user-123"),
    HxSwap("outerHTML"),
    Input(AType("text"), AName("name"), AValue(user.Name)),
    Button(AType("submit"), T("Save")),
)
```

### Live Form Validation

```go
Input(
    AType("email"),
    AName("email"),
    HxGet("/validate/email"),
    HxTrigger("keyup changed delay:500ms"),  // Debounce
    HxTarget("#email-error"),
    HxSync("this:replace"),  // Cancel previous validation
)
Span(AId("email-error"))  // Validation message
```

### Search with Results

```go
Input(
    AType("search"),
    AName("query"),
    APlaceholder("Search..."),
    HxGet("/search"),
    HxTrigger("keyup changed delay:300ms"),
    HxTarget("#results"),
    HxSync("this:replace"),      // Only latest search
    HxIndicator("#search-spinner"),
)
Div(AId("search-spinner"), AClass("htmx-indicator"), T("Searching..."))
Div(AId("results"))
```

### Modal Dialog

```go
// Trigger
Button(
    HxGet("/modal/user-form"),
    HxTarget("body"),
    HxSwap("beforeend"),    // Append modal to body
    T("Open Modal"),
)

// Server returns modal (with close button)
Div(
    AClass("modal"),
    Div(
        AClass("modal-content"),
        Button(
            AClass("close"),
            HxGet("/modal/close"),
            HxTarget("closest .modal"),
            HxSwap("outerHTML swap:200ms"),  // Fade out
            T("×"),
        ),
        // Modal content...
    ),
)
```

### Active Search (Cancel Previous)

```go
Input(
    HxPost("/search"),
    HxTrigger("keyup changed delay:500ms"),
    HxTarget("#results"),
    HxSync("this:abort"),       // Abort current search, start new
    HxIndicator("#spinner"),
)
```

## Integration with plainkit/html

All htmx functions return `html.Global`, making them compatible with any HTML element:

```go
import (
    . "github.com/plainkit/html"
    . "github.com/plainkit/htmx"
)

// Works with any element
Div(HxGet("/data"), HxTarget("#result"))
Button(HxPost("/submit"), T("Submit"))
Form(HxBoost(true), ...)
Input(HxGet("/validate"), HxTrigger("change"))

// Combines with HTML attributes
Button(
    AClass("btn btn-primary"),     // HTML attribute
    AId("submit-btn"),              // HTML attribute
    HxPost("/submit"),              // HTMX attribute
    HxDisabledElt("this"),          // HTMX attribute
    T("Submit"),
)
```

## Serving HTMX JavaScript

Embed htmx.min.js directly in your Go binary:

```go
import "github.com/plainkit/htmx"

// Serve htmx.min.js
http.HandleFunc("/js/htmx.min.js", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/javascript")
    w.Write(htmx.JavaScript())
})

// Include in HTML layout
Script(ASrc("/js/htmx.min.js"))
```

**Benefits:**

- No CDN dependency
- Single binary deployment
- Works offline
- Version control with your app

## Edge Cases & Best Practices

### Prevent Double Submit

```go
Button(
    HxPost("/submit"),
    HxSync("this:drop"),          // Ignore clicks during request
    HxDisabledElt("this"),        // Disable button (UI feedback)
    T("Submit"),
)
```

### Prevent Validation Race Conditions

```go
Input(
    HxGet("/validate"),
    HxTrigger("keyup changed delay:500ms"),
    HxSync("this:replace"),  // Cancel previous validation
)
```

### Empty Input Validation

```go
// Server: Don't show error for empty input (user still typing)
func ValidateEmail(w http.ResponseWriter, r *http.Request) {
    email := r.URL.Query().Get("email")
    if strings.TrimSpace(email) == "" {
        w.WriteHeader(http.StatusOK)  // No content
        return
    }
    // Validate and return error/success message
}
```

### Multiple Element Updates

Use OOB swaps to update cart counter, notification badges, etc. from any response:

```go
// Every response can include OOB updates
Fragment(
    Div(T("Main content")),  // Goes to hx-target
    Span(AId("cart-count"), HxSwapOob("true"), T("5")),  // Updates cart
    Div(AId("notifications"), HxSwapOob("innerHTML"), T("3 new")),  // Updates notifications
)
```

### Accessibility

HTMX updates don't announce to screen readers by default. Add ARIA live regions:

```go
Div(
    AId("results"),
    AAria("live", "polite"),      // Announce changes
    AAria("atomic", "true"),      // Read entire region
)
```

## Common Mistakes

**❌ THE #1 MISTAKE: Returning 4xx/5xx for validation errors**

```go
// ❌ HTMX WILL NOT SWAP THIS - error HTML is sent but never displayed!
func HandleCreate(w http.ResponseWriter, r *http.Request) {
    if invalid {
        errorHTML := RenderForm("Invalid input")
        w.WriteHeader(http.StatusBadRequest) // ❌ HTMX ignores 400!
        w.Write([]byte(errorHTML))
    }
}
```

**✅ Correct**

```go
// ✅ HTMX WILL SWAP THIS - error displays inline
func HandleCreate(w http.ResponseWriter, r *http.Request) {
    isHTMX := r.Header.Get("HX-Request") == "true"

    if invalid {
        errorHTML := RenderForm("Invalid input") // Form with error banner
        if isHTMX {
            w.WriteHeader(http.StatusOK) // ✅ 200 for HTMX
        } else {
            w.WriteHeader(http.StatusBadRequest) // 400 for non-HTMX
        }
        w.Write([]byte(errorHTML))
    }
}
```

**❌ Missing HxEncoding for files**

```go
Form(HxPost("/upload"), Input(AType("file")))  // ❌ Won't work
```

**✅ Correct**

```go
Form(HxPost("/upload"), HxEncoding("multipart/form-data"), Input(AType("file")))
```

**❌ OOB swap without ID**

```go
Span(HxSwapOob("true"), T("content"))  // ❌ No target
```

**✅ Correct**

```go
Span(AId("target"), HxSwapOob("true"), T("content"))
```

**❌ Race conditions in validation**

```go
Input(HxGet("/validate"), HxTrigger("keyup"))  // ❌ Fires too often
```

**✅ Correct**

```go
Input(HxGet("/validate"), HxTrigger("keyup changed delay:500ms"), HxSync("this:replace"))
```

**❌ Pushing URL that doesn't return full page**

```go
Button(HxGet("/partial"), HxPushUrl("true"))  // ❌ Bookmark won't work
```

**✅ Correct**

```go
// Ensure /partial returns full page when accessed directly, or use HxReplaceUrl
```

## Summary

plainkit/htmx provides type-safe Go functions for all HTMX attributes:

1. **HTTP methods** - HxGet, HxPost, HxPut, HxDelete, HxPatch
2. **9 swap strategies** - innerHTML, outerHTML, beforeend, etc. with modifiers
3. **Request sync** - Prevent race conditions with HxSync
4. **OOB swaps** - Update multiple elements from single response
5. **Triggers** - Flexible event syntax with delays, filters, modifiers
6. **Form composition** - HxInclude for cross-field values
7. **File uploads** - HxEncoding for multipart data
8. **History** - HxPushUrl vs HxReplaceUrl
9. **Progressive enhancement** - HxBoost for quick AJAX adoption
10. **Type safety** - All attributes return html.Global, work with any element
11. **Unified endpoints** - Single endpoint pattern for full page + HTMX fragments with automatic query param preservation

**Related skills:**

- skills/plainkit/html - Core HTML generation patterns
- skills/plainkit/webapp - Full architecture and TDD workflow
