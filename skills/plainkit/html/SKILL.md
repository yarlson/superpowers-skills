---
name: Building Type-Safe HTML with plainkit/html
description: Master plainkit/html's function-based HTML generation with compile-time type safety, proper composition patterns, and attribute helpers
when_to_use: When working with plainkit/html library to build Go HTML components - element creation, attributes, children handling, composition. For SVG see skills/plainkit/svg, for icons see skills/plainkit/icons.
version: 2.0.0
---

# Building Type-Safe HTML with plainkit/html

## Overview

plainkit/html is a function-based HTML component library for Go that generates HTML at compile time with zero runtime overhead and full type safety. Each HTML element is a typed function that only accepts valid attributes through interface-based composition.

**Core principle:** Use compile-time type safety to prevent invalid HTML structures before runtime.

**Related skills:**
- skills/plainkit/svg - SVG elements and attributes
- skills/plainkit/icons - Lucide icons library

## Quick Reference

### Element Creation Patterns

```go
// Basic element with attributes and text
Div(AClass("container"), AId("main"), T("Hello"))

// Nested elements
Div(
    AClass("parent"),
    Div(AClass("child"), T("Nested")),
    P(T("Paragraph")),
)

// Multiple children from slice
items := []Node{Li(T("One")), Li(T("Two"))}
Ul(Fragment(toComponents(items)...))
```

### Attribute Helpers

```go
// Global attributes (work with ANY element)
AClass("class-name")              // CSS class
AId("element-id")                 // ID attribute
AData("key", "value")             // data-* attributes
AAria("label", "Main content")    // aria-* attributes
ARole("navigation")               // ARIA role
ACustom("hx-get", "/api")        // Custom attributes (htmx, alpine, etc.)
AStyle("margin: 1rem")           // Inline styles
AOn("click", "handleClick()")    // Event handlers

// Element-specific attributes (type-safe per element)
AType("text")      // Input type
AHref("/path")     // Link href
ARequired()        // Boolean attribute
APlaceholder("text") // Input placeholder
```

### Content and Children

```go
// Text content (auto-escaped for security)
T("Safe <b>text</b>")           // Renders: "Safe &lt;b&gt;text&lt;/b&gt;"

// Unsafe HTML (use with caution)
UnsafeText("<b>Bold</b>")       // Renders: "<b>Bold</b>"

// Single child component
Child(myComponent)              // Or use C() shorthand
C(Div(T("Child")))

// Fragment - multiple children without wrapper
Fragment(
    Div(T("First")),
    P(T("Second")),
)                               // Or use F() shorthand
```

## Core Architecture

### Type System

plainkit/html achieves compile-time type safety through three key types per element:

1. **Attrs struct** - Holds element-specific attributes
2. **Arg interface** - Type constraint that only allows valid options
3. **Apply methods** - Connect options to the element

```go
// Example: Input element architecture
type InputAttrs struct {
    Global GlobalAttrs    // Universal attributes
    Type   string         // Element-specific
    Name   string
    Required bool
    // ...
}

type InputArg interface {
    ApplyInput(*InputAttrs, *[]Component)
}

func Input(args ...InputArg) Node {
    // Constructs type-safe input element
}
```

**Key insight:** The Arg interface ensures only valid attributes can be passed to each element.

### Global vs Element-Specific Attributes

**Global attributes** work with ANY HTML element:
- Implemented via `Global` type
- Includes: class, id, data-*, aria-*, style, events
- Applied through closure pattern

**Element-specific attributes** only work with their element:
- Type-safe through element's Arg interface
- Compile error if used incorrectly
- Examples: `AHref()` only for links, `AType()` for inputs/buttons

```go
// ✅ Valid - type matches
Input(AType("email"), AName("email"))
A(AHref("/home"), T("Home"))

// ❌ Compile error - type mismatch
Input(AHref("/invalid"))  // Href not valid for Input
A(ARequired())            // Required not valid for A
```

## Children and Content Handling

### When to Use What

**T() - Text content:**
- Simple text that needs escaping
- User-generated content
- Any string that might contain HTML

**UnsafeText() - Raw HTML:**
- Pre-sanitized HTML from trusted sources
- Generated HTML from other libraries
- Use with caution

**Child() / C() - Single component:**
- Passing one component as child
- Works with any Component type
- Most common for direct nesting

**Fragment() / F() - Multiple children:**
- Rendering list items without wrapper
- Conditional groups of elements
- When you have []Component to render

### Fragment Pattern

Fragments render children without a wrapper element (like React fragments):

```go
// Without Fragment - creates wrapper div
Div(
    AClass("list"),
    Div(  // ❌ Unnecessary wrapper
        item1,
        item2,
    ),
)

// With Fragment - no wrapper
Div(
    AClass("list"),
    Fragment(  // ✅ Direct children
        item1,
        item2,
    ),
)
```

**Common pattern for dynamic lists:**

```go
var items []Component
for _, todo := range todos {
    items = append(items, Li(T(todo.Title)))
}

Ul(
    AClass("todo-list"),
    Fragment(items...),  // Spread slice as variadic args
)
```

### Type Conversion Pattern

When you have `[]Node` but need `[]Component`:

```go
func toComponents(nodes []Node) []Component {
    components := make([]Component, len(nodes))
    for i, node := range nodes {
        components[i] = node
    }
    return components
}

// Usage
items := []Node{Li(T("One")), Li(T("Two"))}
Ul(Fragment(toComponents(items)...))
```

## Attribute Helper Patterns

### Data Attributes

```go
// Single data attribute
Div(AData("id", "123"))  // renders: data-id="123"

// Multiple data attributes
Div(
    AData("user-id", "42"),
    AData("role", "admin"),
)
```

### ARIA Attributes

```go
// Accessibility attributes
Button(
    AAria("label", "Close dialog"),
    AAria("expanded", "false"),
    T("X"),
)

// Or use ARole for role attribute
Nav(
    ARole("navigation"),
    AAria("label", "Main navigation"),
)
```

### Custom Attributes (htmx, Alpine.js, etc.)

```go
// htmx attributes
Div(
    ACustom("hx-get", "/api/data"),
    ACustom("hx-trigger", "click"),
    ACustom("hx-target", "#result"),
)

// Alpine.js attributes
Div(
    ACustom("x-data", "{ open: false }"),
    ACustom("x-show", "open"),
)
```

### Event Handlers

```go
// Using AOn for events (without "on" prefix)
Button(
    AOn("click", "handleClick()"),
    AOn("mouseover", "highlight()"),
    T("Click me"),
)
// Renders: onclick="handleClick()" onmouseover="highlight()"
```

## Edge Cases and Pitfalls

### ❌ Void Elements Cannot Have Children

```go
// ❌ WRONG - Input is void element
Input(AType("text"), T("content"))  // Compiles but content ignored

// ✅ CORRECT - No children for void elements
Input(AType("text"))

// Other void elements: br, hr, img, link, meta, etc.
```

### ❌ Class Accumulation vs Replacement

```go
// Classes are accumulated (space-separated)
Div(
    AClass("first"),
    AClass("second"),  // ✅ Results in: class="first second"
)

// Not replaced - both classes applied
```

### ❌ Direct Node Passing Without Child()

```go
// When passing Node directly to element, it works
// because Node implements the Apply methods
Div(
    myNode,  // ✅ Works - Node has ApplyDiv method
)

// When passing to different element type
Input(
    myNode,  // ❌ Compile error if myNode doesn't have ApplyInput
)

// Safe: Use Child() to wrap any Component
Div(C(anyComponent))  // ✅ Always works
```

### ✅ Multiple Children Variables

```go
// When you have children in variables
header := Header(T("Title"))
content := Div(T("Content"))
footer := Footer(T("Footer"))

// ❌ WRONG - Can't spread non-slice
Main(header, content, footer...)  // Compile error

// ✅ CORRECT - Pass directly or use Fragment
Main(header, content, footer)  // Direct pass
// OR
Main(Fragment(header, content, footer))  // Fragment wrapper
```

### ✅ Conditional Rendering

```go
// Build children conditionally
var children []Component
children = append(children, Header(T("Title")))

if showContent {
    children = append(children, Div(T("Content")))
}

if showFooter {
    children = append(children, Footer(T("Footer")))
}

Main(Fragment(children...))
```

## Function Composition Patterns

### Composable Components

Build reusable components by composing smaller ones:

```go
func Card(title, content string, actions ...Component) Node {
    return Div(
        AClass("card"),
        Header(
            AClass("card-header"),
            H3(T(title)),
        ),
        Div(
            AClass("card-body"),
            P(T(content)),
        ),
        Footer(
            AClass("card-footer"),
            Fragment(actions...),  // Spread actions
        ),
    )
}

// Usage
Card(
    "Welcome",
    "Get started with plainkit",
    Button(AType("button"), T("Learn More")),
    Button(AType("button"), T("Docs")),
)
```

### Props Pattern (Advanced)

Create typed component props that implement element interfaces:

```go
type ButtonProps struct {
    Variant string
    Size    string
    FullWidth bool
}

// Implement ButtonArg interface
func (p ButtonProps) ApplyButton(attrs *ButtonAttrs, kids *[]Component) {
    // Build classes based on props
    className := buildButtonClasses(p.Variant, p.Size, p.FullWidth)
    AClass(className).ApplyButton(attrs, kids)
}

// Usage
Button(
    ButtonProps{Variant: "primary", Size: "lg"},
    T("Click me"),
)
```

## Component Rendering

### Render to String

```go
component := Div(AClass("test"), T("Hello"))
html := Render(component)  // Returns HTML string

// In HTTP handlers
func handler(w http.ResponseWriter, r *http.Request) {
    page := Html(
        Head(Title(T("Page"))),
        Body(H1(T("Hello"))),
    )

    w.Header().Set("Content-Type", "text/html; charset=utf-8")
    fmt.Fprint(w, "<!DOCTYPE html>\n")
    fmt.Fprint(w, Render(page))
}
```

### Asset System (CSS/JS)

Components can declare CSS and JavaScript dependencies:

```go
component := Div(T("Content")).WithAssets(
    ".my-class { color: blue; }",  // CSS
    "console.log('loaded')",        // JS
    "unique-name",                  // Deduplication key
)
```

## Testing Components

```go
func TestUserCard(t *testing.T) {
    card := Div(
        AClass("user-card"),
        H3(T("John Doe")),
        P(T("john@example.com")),
    )

    html := Render(card)

    if !strings.Contains(html, "John Doe") {
        t.Error("Missing user name")
    }

    if !strings.Contains(html, `class="user-card"`) {
        t.Error("Missing class")
    }
}
```

## Common Patterns Checklist

When building with plainkit/html:

- [ ] Use `T()` for text content (auto-escaped)
- [ ] Use global attributes (`AClass`, `AId`, etc.) for any element
- [ ] Use element-specific attributes only where valid
- [ ] Use `Fragment()` for multiple children without wrapper
- [ ] Convert `[]Node` to `[]Component` before spreading
- [ ] Build conditional children in slice, then spread to `Fragment()`
- [ ] Use `Child()` / `C()` to safely wrap any component
- [ ] Avoid children on void elements (input, br, hr, img, etc.)
- [ ] Use `AData()` for data-* attributes
- [ ] Use `AAria()` for aria-* attributes
- [ ] Use `ACustom()` for framework-specific attributes (htmx, alpine)
- [ ] Test components by rendering to string

**For SVG:** See skills/plainkit/svg
**For Icons:** See skills/plainkit/icons

## Anti-Patterns to Avoid

❌ **Don't** try to pass children to void elements
❌ **Don't** use string concatenation for classes (use multiple `AClass()`)
❌ **Don't** use element-specific attrs on wrong elements
❌ **Don't** forget to escape user input (use `T()` not `UnsafeText()`)
❌ **Don't** spread non-slice variables (`...` only works on slices)
❌ **Don't** create wrapper divs unnecessarily (use `Fragment()`)

## Summary

plainkit/html provides compile-time type safety for HTML generation through:

1. **Type-safe attributes** - Element-specific Arg interfaces
2. **Global attributes** - Work with any element via Global type
3. **Flexible children** - Direct pass, Child(), or Fragment()
4. **Safe text** - Auto-escaped T() vs unsafe UnsafeText()
5. **Composition** - Build reusable components through function composition

The library enforces correctness at compile time, preventing invalid HTML structures before runtime.

**Related skills:**
- skills/plainkit/svg - SVG elements and attributes with composition limitations
- skills/plainkit/icons - 1000+ Lucide icons with sensible defaults
