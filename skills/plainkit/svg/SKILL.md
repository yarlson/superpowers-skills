---
name: SVG Elements with plainkit/html
description: SVG element creation and attributes in plainkit/html with composition limitations and workarounds
when_to_use: When creating SVG graphics with plainkit/html - basic shapes, paths, attributes. For pre-built icons see skills/plainkit/icons. For core HTML see skills/plainkit/html.
version: 1.0.0
---

# SVG Elements with plainkit/html

## Overview

plainkit/html provides SVG support with the same type-safe approach as HTML elements. SVG elements are prefixed with `Svg` to avoid naming conflicts. However, the current implementation has **limited support for nested SVG composition**.

**Core principle:** Use basic SVG elements directly under root `Svg()`, or use skills/plainkit/icons for complex compositions.

**Related skills:**
- skills/plainkit/html - Core HTML patterns, attributes, and composition
- skills/plainkit/icons - Pre-built Lucide icons for complex SVG needs

## Quick Reference

### SVG Elements

```go
// Basic SVG elements (fully supported)
Svg(...)           // Root SVG element
SvgCircle(...)     // Circle
SvgPath(...)       // Path
SvgRect(...)       // Rectangle
SvgLine(...)       // Line
SvgPolygon(...)    // Polygon
SvgPolyline(...)   // Polyline
SvgEllipse(...)    // Ellipse

// Limited support elements (see limitations below)
SvgG(...)          // Group
SvgText(...)       // Text
```

### SVG Attributes

```go
// Geometry
ACx("50"), ACy("50")    // Circle center
AR("40")                // Circle radius
AX("0"), AY("0")        // Position
AD("M10 10 L90 90")     // Path data

// Visual
AFill("blue")           // Fill color
AStroke("red")          // Stroke color
AStrokeWidth("2")       // Stroke width
AViewBox("0 0 24 24")   // ViewBox
AOpacity("0.8")         // Opacity

// Transform
ATransform("rotate(45)") // Transform
```

## SVG Element Creation

### Basic SVG with Circle

```go
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
```

### SVG Path Example

```go
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
```

## SVG-Specific Attributes

### Geometry Attributes

```go
// Circle/Ellipse
ACx("50")           // cx (center x)
ACy("50")           // cy (center y)
AR("40")            // r (radius for circle)
ARx("30")           // rx (x-axis radius for ellipse)
ARy("20")           // ry (y-axis radius for ellipse)

// Position
AX("10")            // x position
AY("10")            // y position

// Line
AX1("0"), AY1("0")  // line start
AX2("100"), AY2("100") // line end

// Path
AD("M 10 10 L 90 90")  // d (path data)
```

### Visual Attributes

```go
// Fill
AFill("red")           // fill color
AFillOpacity("0.5")    // fill opacity
AFillRule("evenodd")   // fill rule

// Stroke
AStroke("blue")        // stroke color
AStrokeWidth("2")      // stroke width
AStrokeOpacity("0.8")  // stroke opacity
AStrokeLinecap("round") // stroke line cap (butt, round, square)
AStrokeLinejoin("round") // stroke line join (miter, round, bevel)
AStrokeDasharray("5,5") // stroke dash pattern

// Overall
AOpacity("0.8")        // overall opacity
```

### Transform and ViewBox

```go
ATransform("rotate(45)")           // Rotate 45 degrees
ATransform("translate(10, 20)")    // Move by x,y
ATransform("scale(2)")             // Scale 2x
ATransform("matrix(1,0,0,1,0,0)")  // Full transformation matrix

AViewBox("0 0 100 100")  // ViewBox coordinate system
APreserveAspectRatio("xMidYMid meet") // Aspect ratio handling
```

### Namespace

```go
// For root svg element
AXmlns("http://www.w3.org/2000/svg") // SVG namespace
```

## SVG Type System

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

## Composition Limitations

**CRITICAL:** The current plainkit/html implementation has limited support for nested SVG elements.

### What Doesn't Work

```go
// ❌ This DOES NOT compile - nested SVG elements
chart := Svg(
    AViewBox("0 0 250 150"),
    SvgG(  // ❌ Cannot nest children in SvgG
        SvgRect(...),  // Node doesn't have ApplyG method
        SvgRect(...),
    ),
)

// ❌ This DOES NOT compile - text in SVG
label := SvgText(
    AX("50"), AY("50"),
    T("Label"),  // ❌ TxtOpt doesn't have ApplyText method
)
```

### Why This Happens

The library is missing `Apply*` methods for SVG element composition:
- `Node` doesn't implement `ApplyG`, `ApplyText`, etc.
- `TxtOpt` doesn't implement `ApplyText`
- Child SVG elements cannot be passed to parent SVG elements

### Workarounds

**Option 1: Use Lucide Icons Library** (Recommended)

For complex SVG with groups, text, and nested elements:

```go
import "github.com/plainkit/icons/lucide"

// Pre-built, tested icons
icon := lucide.Heart(lucide.Size("24"))
menu := lucide.Menu(html.AClass("icon"))
```

See skills/plainkit/icons for full documentation.

**Option 2: Use UnsafeText()**

For custom complex SVG:

```go
svg := Svg(
    AViewBox("0 0 100 100"),
    AWidth("100"),
    AHeight("100"),
    UnsafeText(`
        <g transform="translate(10, 10)">
            <rect x="0" y="0" width="20" height="80" fill="red"/>
            <text x="10" y="90" text-anchor="middle">Red</text>
        </g>
    `),
)
```

**Option 3: Multiple Root-Level Elements**

For simple cases, add elements directly under `Svg()`:

```go
// ✅ This works - direct children of Svg()
icon := Svg(
    AViewBox("0 0 100 100"),
    SvgCircle(ACx("25"), ACy("50"), AR("20"), AFill("red")),
    SvgCircle(ACx("75"), ACy("50"), AR("20"), AFill("blue")),
)
```

## Common Patterns

### Simple Icons

```go
func CheckIcon() html.Node {
    return Svg(
        AViewBox("0 0 24 24"),
        AWidth("20"),
        AHeight("20"),
        AClass("icon"),
        SvgPath(
            AD("M20 6L9 17l-5-5"),
            AFill("none"),
            AStroke("currentColor"),
            AStrokeWidth("2"),
            AStrokeLinecap("round"),
            AStrokeLinejoin("round"),
        ),
    )
}
```

### Responsive SVG

```go
func ResponsiveSvg(content ...html.SvgArg) html.Node {
    defaults := []html.SvgArg{
        AViewBox("0 0 100 100"),
        AWidth("100%"),
        AHeight("100%"),
        APreserveAspectRatio("xMidYMid meet"),
    }
    return Svg(append(defaults, content...)...)
}
```

## Common Patterns Checklist

- [ ] Use `Svg` prefix for all SVG elements
- [ ] Set `AViewBox()` for scalable graphics
- [ ] Use `AXmlns()` on root `Svg()` element
- [ ] Keep SVG composition flat (avoid nesting)
- [ ] Use skills/plainkit/icons for complex SVG
- [ ] Use `UnsafeText()` for custom complex SVG
- [ ] Use `currentColor` for stroke/fill to inherit text color
- [ ] Test SVG rendering in target browsers

## Summary

plainkit/html provides type-safe SVG element creation with:

1. **Basic SVG elements** - Circle, Path, Rect, Line, Polygon, Polyline, Ellipse
2. **Comprehensive attributes** - Geometry, visual, transform, viewBox
3. **Type safety** - Compile-time validation of attributes
4. **Composition limitations** - Nested SVG elements not supported

**For complex SVG:**
- Use skills/plainkit/icons for pre-built icons
- Use `UnsafeText()` for custom nested SVG
- Keep composition flat under root `Svg()` element

**Related skills:**
- skills/plainkit/html - Core patterns and composition
- skills/plainkit/icons - 1000+ pre-built Lucide icons
