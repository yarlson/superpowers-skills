---
name: Building Type-Safe HTML with plainkit/html
description: Master plainkit/html's function-based HTML generation with compile-time type safety, proper composition patterns, and attribute helpers
when_to_use: When working with plainkit/html library to build Go HTML components - includes element creation, attributes, children handling, and composition
version: 1.0.0
---

# Building Type-Safe HTML with plainkit/html

## Overview

plainkit/html is a function-based HTML component library for Go that generates HTML at compile time with zero runtime overhead and full type safety. Each HTML element is a typed function that only accepts valid attributes through interface-based composition.

**Core principle:** Use compile-time type safety to prevent invalid HTML structures before runtime.

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

### SVG Elements and Attributes

```go
// SVG elements (prefixed with Svg)
Svg(...)           // Root SVG element
SvgCircle(...)     // Circle
SvgPath(...)       // Path
SvgRect(...)       // Rectangle
SvgG(...)          // Group
SvgText(...)       // Text (for labels)
SvgPolyline(...)   // Polyline (for line charts)

// SVG geometry attributes
ACx("50"), ACy("50")    // Circle center
AR("40")                // Circle radius
AD("M10 10 L90 90")     // Path data
AX("0"), AY("0")        // Position

// SVG visual attributes
AFill("blue")           // Fill color
AStroke("red")          // Stroke color
AStrokeWidth("2")       // Stroke width
AViewBox("0 0 24 24")   // ViewBox

// SVG text attributes
ATextAnchor("middle")   // Text alignment
AFontSize("14")         // Font size
AFontFamily("Arial")    // Font family

// Lucide icons
lucide.Heart(lucide.Size("24"))
lucide.Menu(html.AClass("icon"))
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
- SVG content
- Generated HTML from other libraries

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
- [ ] Use `Svg` prefix for SVG elements (`SvgCircle`, `SvgPath`, etc.)
- [ ] Use SVG-specific attributes (`ACx`, `ACy`, `AR`, `AD`, `AFill`, `AStroke`)
- [ ] Use `lucide.IconName()` for pre-built icons from Lucide library
- [ ] Use `lucide.Size()` for uniform icon sizing
- [ ] Test components by rendering to string

## Anti-Patterns to Avoid

❌ **Don't** try to pass children to void elements
❌ **Don't** use string concatenation for classes (use multiple `AClass()`)
❌ **Don't** use element-specific attrs on wrong elements
❌ **Don't** forget to escape user input (use `T()` not `UnsafeText()`)
❌ **Don't** spread non-slice variables (`...` only works on slices)
❌ **Don't** create wrapper divs unnecessarily (use `Fragment()`)

## SVG Support

### SVG Element Naming

SVG elements are prefixed with `Svg` to avoid naming conflicts with HTML elements:

```go
// SVG elements use Svg prefix
Svg(...)           // <svg> element
SvgCircle(...)     // <circle> element
SvgPath(...)       // <path> element
SvgRect(...)       // <rect> element
SvgG(...)          // <g> (group) element
SvgLine(...)       // <line> element
SvgPolygon(...)    // <polygon> element
SvgPolyline(...)   // <polyline> element
SvgEllipse(...)    // <ellipse> element
SvgText(...)       // <text> element (for labels)
SvgTitle(...)      // <title> element (accessibility)
```

### SVG-Specific Attributes

SVG elements use dedicated attribute helpers for SVG-specific attributes:

```go
// Geometry attributes
ACx("50")           // cx (center x for circle/ellipse)
ACy("50")           // cy (center y for circle/ellipse)
AR("40")            // r (radius for circle)
ARx("30")           // rx (x-axis radius)
ARy("20")           // ry (y-axis radius)
AX("10")            // x position
AY("10")            // y position
AX1("0"), AY1("0")  // line start
AX2("100"), AY2("100") // line end

// Path data
AD("M 10 10 L 90 90")  // d (path data)

// Visual attributes
AFill("red")           // fill color
AFillOpacity("0.5")    // fill opacity
AStroke("blue")        // stroke color
AStrokeWidth("2")      // stroke width
AStrokeLinecap("round") // stroke line cap
AStrokeLinejoin("round") // stroke line join
AStrokeDasharray("5,5") // stroke dash pattern
AOpacity("0.8")        // overall opacity

// Text attributes (for SvgText)
ATextAnchor("middle")  // text alignment (start, middle, end)
AFontSize("14")        // font size
AFontFamily("Arial")   // font family
AFontWeight("bold")    // font weight

// Transform and viewBox
ATransform("rotate(45)") // transform
AViewBox("0 0 100 100")  // viewBox
APreserveAspectRatio("xMidYMid meet") // aspect ratio

// Namespace (for root svg element)
AXmlns("http://www.w3.org/2000/svg") // xmlns
```

### SVG Usage Pattern

```go
// Basic SVG with circle
icon := Svg(
    AViewBox("0 0 100 100"),
    AWidth("100"),
    AHeight("100"),
    AXmlns("http://www.w3.org/2000/svg"),
    SvgCircle(
        ACx("50"),
        ACy("50"),
        AR("40"),
        AFill("blue"),
        AStroke("black"),
        AStrokeWidth("2"),
    ),
)

// SVG path example
checkmark := Svg(
    AViewBox("0 0 24 24"),
    AWidth("24"),
    AHeight("24"),
    SvgPath(
        AD("M20 6L9 17l-5-5"),
        AFill("none"),
        AStroke("currentColor"),
        AStrokeWidth("2"),
        AStrokeLinecap("round"),
        AStrokeLinejoin("round"),
    ),
)

// Complex SVG with groups and text labels
chart := Svg(
    AViewBox("0 0 250 150"),
    AXmlns("http://www.w3.org/2000/svg"),
    // Bars group
    SvgG(
        AClass("bars"),
        ATransform("translate(20, 10)"),
        SvgRect(AX("0"), AY("20"), AWidth("40"), AHeight("80"), AFill("red")),
        SvgRect(AX("60"), AY("40"), AWidth("40"), AHeight("60"), AFill("green")),
        SvgRect(AX("120"), AY("60"), AWidth("40"), AHeight("40"), AFill("blue")),
    ),
    // Labels group
    SvgG(
        AClass("labels"),
        SvgText(
            AX("40"), AY("125"),
            ATextAnchor("middle"),
            AFontSize("12"),
            T("Red"),
        ),
        SvgText(
            AX("100"), AY("125"),
            ATextAnchor("middle"),
            AFontSize("12"),
            T("Green"),
        ),
        SvgText(
            AX("160"), AY("125"),
            ATextAnchor("middle"),
            AFontSize("12"),
            T("Blue"),
        ),
    ),
)
```

### SVG Type System

SVG elements have their own Arg interfaces, separate from HTML elements:

```go
// Each SVG element has specific Arg interface
type SvgCircleArg interface {
    ApplyCircle(*SvgCircleAttrs, *[]Component)
}

type SvgPathArg interface {
    ApplyPath(*SvgPathAttrs, *[]Component)
}

// Global attributes work with SVG elements
Svg(
    AClass("icon"),              // ✅ Global attribute
    AId("my-svg"),               // ✅ Global attribute
    AData("icon-type", "user"),  // ✅ Global attribute
    AFill("blue"),               // ✅ SVG-specific attribute
)
```

## Lucide Icons Library

PlainKit includes a comprehensive icon library at `github.com/plainkit/icons/lucide` with 1000+ icons from Lucide.

### Basic Usage

```go
import (
    "github.com/plainkit/html"
    "github.com/plainkit/icons/lucide"
)

// Use icon with default size (24x24)
icon := lucide.Heart()

// Customize with attributes
icon := lucide.Heart(
    lucide.Size("16"),           // Resize to 16x16
    html.AClass("text-red-500"), // Add CSS class
)

// Icons work with any SvgArg
icon := lucide.Activity(
    lucide.Size("32"),
    html.AStroke("blue"),
    html.AStrokeWidth("3"),
    html.AClass("animate-pulse"),
)
```

### Size Helper

The `Size()` helper uniformly sets both width and height:

```go
lucide.Star(lucide.Size("20"))  // 20x20
lucide.Heart(lucide.Size("48")) // 48x48

// Without Size(), set individually:
lucide.Star(html.AWidth("20"), html.AHeight("20"))
```

### Icon Defaults

All Lucide icons come with sensible defaults:
- **Size**: 24x24 pixels
- **ViewBox**: "0 0 24 24"
- **Fill**: "none"
- **Stroke**: "currentColor" (inherits text color)
- **Stroke Width**: "2"
- **Stroke Linecap**: "round"
- **Stroke Linejoin**: "round"
- **Class**: "lucide lucide-{icon-name}"

```go
// Icon inherits text color via currentColor
Div(
    AClass("text-blue-500"),  // Text color
    lucide.Heart(),           // Icon will be blue
)

// Override defaults
lucide.Heart(
    html.AStroke("red"),      // Override stroke color
    html.AFill("pink"),       // Override fill
    html.AStrokeWidth("3"),   // Override stroke width
)
```

### Available Icons

The library includes 1000+ icons. Common examples:

```go
// UI & Navigation
lucide.Menu()
lucide.ChevronDown()
lucide.ArrowRight()
lucide.Home()
lucide.Search()
lucide.Settings()

// Actions
lucide.Plus()
lucide.Minus()
lucide.X()
lucide.Check()
lucide.Edit()
lucide.Trash()

// Media
lucide.Play()
lucide.Pause()
lucide.Volume()
lucide.Image()
lucide.Video()

// Communication
lucide.Mail()
lucide.MessageSquare()
lucide.Phone()
lucide.Bell()

// Files & Folders
lucide.File()
lucide.Folder()
lucide.Download()
lucide.Upload()

// Business
lucide.ShoppingCart()
lucide.CreditCard()
lucide.DollarSign()
lucide.TrendingUp()
```

### Integration with Components

```go
// Icon button component
func IconButton(icon html.Node, label string) html.Node {
    return Button(
        AType("button"),
        AClass("icon-button"),
        AAria("label", label),
        icon,
    )
}

// Usage
deleteBtn := IconButton(
    lucide.Trash(lucide.Size("20")),
    "Delete item",
)

// List with icons
items := []string{"Home", "About", "Contact"}
icons := []html.Node{
    lucide.Home(),
    lucide.Info(),
    lucide.Mail(),
}

var listItems []html.Component
for i, item := range items {
    listItems = append(listItems, Li(
        AClass("flex items-center gap-2"),
        icons[i],
        T(item),
    ))
}

Nav(Ul(Fragment(listItems...)))
```

## Summary

plainkit/html provides compile-time type safety for HTML generation through:

1. **Type-safe attributes** - Element-specific Arg interfaces
2. **Global attributes** - Work with any element via Global type
3. **Flexible children** - Direct pass, Child(), or Fragment()
4. **Safe text** - Auto-escaped T() vs unsafe UnsafeText()
5. **Composition** - Build reusable components through function composition
6. **SVG support** - Full SVG element and attribute support with Svg prefix
7. **Icon library** - 1000+ Lucide icons with sensible defaults

The library enforces correctness at compile time, preventing invalid HTML structures before runtime.
