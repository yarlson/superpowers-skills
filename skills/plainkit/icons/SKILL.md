---
name: Lucide Icons for plainkit/html
description: Pre-built SVG icons from Lucide library - 1000+ icons with sensible defaults, Size helper, and customization options
when_to_use: When you need icons in plainkit/html applications - navigation, actions, media, communication, files, business icons. For basic SVG see skills/plainkit/svg. For core HTML see skills/plainkit/html.
version: 1.0.0
---

# Lucide Icons for plainkit/html

## Overview

PlainKit includes a comprehensive icon library at `github.com/plainkit/icons/lucide` with 1000+ icons from Lucide. All icons are pre-built, tested SVG components with sensible defaults.

**Core principle:** Use Lucide icons for all SVG icons instead of building custom SVG - they're pre-tested, accessible, and consistent.

**Related skills:**
- skills/plainkit/html - Core HTML patterns and composition
- skills/plainkit/svg - Custom SVG elements and attributes

## Quick Reference

```go
import (
    "github.com/plainkit/html"
    "github.com/plainkit/icons/lucide"
)

// Basic usage
lucide.Heart()                      // Default 24x24
lucide.Menu(lucide.Size("20"))      // Custom size
lucide.Settings(html.AClass("icon")) // With CSS class

// Multiple attributes
lucide.Activity(
    lucide.Size("32"),
    html.AStroke("blue"),
    html.AStrokeWidth("3"),
    html.AClass("animate-pulse"),
)
```

## Basic Usage

### Default Icons

```go
// Use icon with default size (24x24)
icon := lucide.Heart()

// Icons return html.Node, use anywhere
Button(
    AType("button"),
    lucide.Trash(),
    T("Delete"),
)
```

### Customizing Icons

```go
// Resize with Size() helper
icon := lucide.Heart(lucide.Size("16"))  // 16x16

// Add CSS classes
icon := lucide.Menu(html.AClass("text-gray-500"))

// Multiple customizations
icon := lucide.Activity(
    lucide.Size("32"),
    html.AStroke("blue"),
    html.AStrokeWidth("3"),
    html.AClass("animate-pulse"),
)
```

## Size Helper

The `Size()` helper uniformly sets both width and height:

```go
lucide.Star(lucide.Size("20"))  // 20x20
lucide.Heart(lucide.Size("48")) // 48x48

// Without Size(), set individually:
lucide.Star(html.AWidth("20"), html.AHeight("20"))
```

**Why use Size():** Ensures icons stay square and properly sized.

## Icon Defaults

All Lucide icons come with sensible defaults:

| Property | Default Value |
|----------|---------------|
| **Size** | 24x24 pixels |
| **ViewBox** | "0 0 24 24" |
| **Fill** | "none" |
| **Stroke** | "currentColor" (inherits text color) |
| **Stroke Width** | "2" |
| **Stroke Linecap** | "round" |
| **Stroke Linejoin** | "round" |
| **Class** | "lucide lucide-{icon-name}" |

### Color Inheritance

Icons use `currentColor` for stroke, so they inherit the parent's text color:

```go
// Icon will be blue
Div(
    AClass("text-blue-500"),  // Text color
    lucide.Heart(),           // Icon inherits blue
)

// Override with custom color
lucide.Heart(html.AStroke("red"))
```

### Overriding Defaults

```go
lucide.Heart(
    html.AStroke("red"),      // Override stroke color
    html.AFill("pink"),       // Override fill
    html.AStrokeWidth("3"),   // Override stroke width
    lucide.Size("32"),        // Override size
)
```

## Available Icons

The library includes 1000+ icons. Common categories:

### UI & Navigation

```go
lucide.Menu()          // Hamburger menu
lucide.ChevronDown()   // Dropdown indicator
lucide.ChevronUp()     // Expand indicator
lucide.Arrow Right()    // Next/forward
lucide.ArrowLeft()     // Back/previous
lucide.Home()          // Home page
lucide.Search()        // Search
lucide.Settings()      // Settings/config
lucide.MoreVertical()  // More options (⋮)
lucide.MoreHorizontal() // More options (⋯)
```

### Actions

```go
lucide.Plus()          // Add/create
lucide.Minus()         // Remove/subtract
lucide.X()             // Close/dismiss
lucide.Check()         // Confirm/success
lucide.Edit()          // Edit/modify
lucide.Trash()         // Delete
lucide.Save()          // Save
lucide.Copy()          // Copy
lucide.ExternalLink()  // Open in new tab
```

### Media

```go
lucide.Play()          // Play video/audio
lucide.Pause()         // Pause
lucide.Volume()        // Sound
lucide.VolumeX()       // Muted
lucide.Image()         // Image/photo
lucide.Video()         // Video
lucide.Camera()        // Camera/photo
```

### Communication

```go
lucide.Mail()          // Email
lucide.MessageSquare() // Chat/message
lucide.MessageCircle() // Comment
lucide.Phone()         // Phone call
lucide.Bell()          // Notifications
lucide.Send()          // Send message
```

### Files & Folders

```go
lucide.File()          // Generic file
lucide.FileText()      // Text document
lucide.Folder()        // Folder
lucide.FolderOpen()    // Open folder
lucide.Download()      // Download
lucide.Upload()        // Upload
```

### Business

```go
lucide.ShoppingCart()  // Shopping cart
lucide.CreditCard()    // Payment
lucide.DollarSign()    // Money/pricing
lucide.TrendingUp()    // Growth/analytics
lucide.BarChart()      // Charts/reports
lucide.Users()         // Team/people
```

### Status & Alerts

```go
lucide.AlertCircle()   // Warning
lucide.AlertTriangle() // Error
lucide.Info()          // Information
lucide.CheckCircle()   // Success
lucide.XCircle()       // Error/failed
```

**Note:** For complete icon list, see [Lucide icon directory](https://lucide.dev/icons)

## Integration Patterns

### Icon Button Component

```go
func IconButton(icon html.Node, label string, onclick string) html.Node {
    return Button(
        AType("button"),
        AClass("icon-button"),
        AAria("label", label),
        AOn("click", onclick),
        icon,
    )
}

// Usage
deleteBtn := IconButton(
    lucide.Trash(lucide.Size("20")),
    "Delete item",
    "handleDelete()",
)
```

### Navigation with Icons

```go
func NavLink(icon html.Node, text, href string) html.Node {
    return A(
        AHref(href),
        AClass("nav-link flex items-center gap-2"),
        icon,
        T(text),
    )
}

// Usage
nav := Nav(
    NavLink(lucide.Home(), "Home", "/"),
    NavLink(lucide.Search(), "Search", "/search"),
    NavLink(lucide.Settings(), "Settings", "/settings"),
)
```

### List with Icons

```go
items := []struct {
    Icon html.Node
    Text string
}{
    {lucide.Home(), "Home"},
    {lucide.Info(), "About"},
    {lucide.Mail(), "Contact"},
}

var listItems []html.Component
for _, item := range items {
    listItems = append(listItems, Li(
        AClass("flex items-center gap-2"),
        item.Icon,
        T(item.Text),
    ))
}

Ul(Fragment(listItems...))
```

### Icon with Badge

```go
func IconWithBadge(icon html.Node, count int) html.Node {
    return Div(
        AClass("relative"),
        icon,
        Span(
            AClass("badge"),
            T(fmt.Sprintf("%d", count)),
        ),
    )
}

// Usage
notification := IconWithBadge(lucide.Bell(lucide.Size("20")), 5)
```

### Loading State

```go
func LoadingButton(text string, loading bool) html.Node {
    var icon html.Node
    if loading {
        icon = lucide.Loader2(
            lucide.Size("16"),
            html.AClass("animate-spin"),
        )
    }

    return Button(
        AType("button"),
        ADisabled(loading),
        Fragment(
            icon,
            T(text),
        ),
    )
}
```

## Common Patterns Checklist

- [ ] Use Lucide icons instead of custom SVG
- [ ] Use `lucide.Size()` for consistent sizing
- [ ] Let icons inherit color with `currentColor`
- [ ] Add `AAria("label", "...")` for icon-only buttons
- [ ] Use descriptive icon names (lucide.Trash not lucide.Delete)
- [ ] Combine icons with text for clarity
- [ ] Test icon visibility in dark/light modes

## Summary

Lucide icons for plainkit/html provide:

1. **1000+ pre-built icons** - Comprehensive coverage for all use cases
2. **Sensible defaults** - 24x24, currentColor, rounded strokes
3. **Size helper** - Uniform sizing with `lucide.Size()`
4. **Customizable** - Override any SVG attribute
5. **Type-safe** - Full compile-time validation
6. **Tested** - Pre-built, working SVG components

**Related skills:**
- skills/plainkit/html - Core HTML patterns and composition
- skills/plainkit/svg - Custom SVG for special cases
