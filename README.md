# ZUI FRAMEWORK: COMPLETE SPECIFICATION V6

**Version:** 6.0.0 - Production Ready Architecture
**Target Zig Version:** 0.15.x
**Target Platform:** Fermyon Spin (wasm32-freestanding + wasm32-wasi)
**Rendering Modes:** SPA, SSR, SSG
**Status:** Final Specification with Complete Architecture

---

## EXECUTIVE SUMMARY

Zui v6 represents a complete architectural revision with:

- **Complete Virtual DOM** with efficient diffing and reconciliation
- **Context-based architecture** eliminating mutable global state
- **Type-safe routing** with compile-time validation
- **Real CSS extraction** via static analysis at compile-time
- **Effects system** for asynchronous operations and side effects
- **Enhanced JS runtime** with error handling and update batching
- **Zero CSS runtime overhead** - all styles compiled at compile-time
- **Multiple rendering modes** - SPA, SSR, and SSG support
- **Fermyon Spin integration** - edge-ready deployment platform

---

## ARCHITECTURE DIAGRAM

```
┌─────────────────────────────────────────────────────────────────┐
│                      FERMYON SPIN PLATFORM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 ZUI APPLICATION                          │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │   Router    │  │   Effects   │  │    State    │      │    │
│  │  │ (Type-safe) │  │   System    │  │  Management │      │    │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │    │
│  │         │                │                │              │    │
│  │         └────────────────┼────────────────┘              │    │
│  │                          │                               │    │
│  │                    ┌─────▼─────┐                         │    │
│  │                    │ AppContext│                         │    │
│  │                    │(No globals)                         │    │
│  │                    └─────┬─────┘                         │    │
│  │                          │                               │    │
│  │  ┌───────────────────────┼───────────────────────┐      │    │
│  │  │              Virtual DOM                       │      │    │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────────┐    │      │    │
│  │  │  │ Elements│  │ Diffing │  │Reconciliation│    │      │    │
│  │  │  └─────────┘  └─────────┘  └─────────────┘    │      │    │
│  │  └───────────────────────┬───────────────────────┘      │    │
│  │                          │                               │    │
│  │  ┌───────────────────────┼───────────────────────┐      │    │
│  │  │           ZSS (Zig Style Sheets)               │      │    │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────────┐    │      │    │
│  │  │  │  Types  │  │ Builder │  │  Extractor  │    │      │    │
│  │  │  └─────────┘  └─────────┘  └─────────────┘    │      │    │
│  │  └───────────────────────────────────────────────┘      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│  ┌───────────────────────────┼───────────────────────────────┐  │
│  │                    RENDERING MODES                         │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │  │
│  │  │     SPA     │  │     SSR     │  │     SSG     │        │  │
│  │  │  (Browser)  │  │  (Server)   │  │ (Build-time)│        │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
├──────────────────────────────┼───────────────────────────────────┤
│  ┌───────────────────────────┼───────────────────────────────┐  │
│  │  Static Files  │  SSR/API WASM   │  JS Runtime (zui.js)   │  │
│  │  (SPA/SSG)     │  (wasm32-wasi)  │  (Browser hydration)   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## SPECIFICATIONS INDEX

### Core Infrastructure

| Document | Description |
|-----------|-------------|
| [00-overview.md](spec/00-overview.md) | Project objectives, principles and constraints |
| [01-core-infrastructure.md](spec/01-core-infrastructure.md) | Memory management and utilities |
| [02-context-system.md](spec/02-context-system.md) | Dependency injection and context passing |

### Virtual DOM and Rendering

| Document | Description |
|-----------|-------------|
| [03-virtual-dom.md](spec/03-virtual-dom.md) | Complete VDOM with diffing and reconciliation |
| [04-html-dsl.md](spec/04-html-dsl.md) | Type-safe HTML element construction |

### Style System (ZSS)

| Document | Description |
|-----------|-------------|
| [05-css-types.md](spec/05-css-types.md) | Type-safe CSS value definitions |
| [06-css-rules.md](spec/06-css-rules.md) | Style rules and properties |
| [07-css-builder.md](spec/07-css-builder.md) | Atomic CSS generation |
| [08-css-extraction.md](spec/08-css-extraction.md) | Static analysis and compile-time extraction |
| [09-css-theme.md](spec/09-css-theme.md) | Theme system |

### Application Features

| Document | Description |
|-----------|-------------|
| [10-routing.md](spec/10-routing.md) | Type-safe routing system |
| [11-effects-system.md](spec/11-effects-system.md) | Side effects and async operations |
| [12-state-management.md](spec/12-state-management.md) | Application state management |

### Runtime and Build

| Document | Description |
|-----------|-------------|
| [13-wasm-runtime.md](spec/13-wasm-runtime.md) | WASM exports and imports |
| [14-js-runtime.md](spec/14-js-runtime.md) | Enhanced JavaScript runtime |
| [15-build-system.md](spec/15-build-system.md) | Build configuration and steps |

### Deployment

| Document | Description |
|-----------|-------------|
| [18-fermyon-spin.md](spec/18-fermyon-spin.md) | Fermyon Spin deployment and rendering modes |

### Developer Experience

| Document | Description |
|-----------|-------------|
| [19-developer-experience.md](spec/19-developer-experience.md) | Hot reload, dev server, debugging, tooling |
| [20-browser-apis.md](spec/20-browser-apis.md) | Browser API integration (storage, crypto, clipboard, etc.) |
| [21-tailwind-integration.md](spec/21-tailwind-integration.md) | Type-safe Tailwind CSS integration |

### Advanced Features

| Document | Description |
|-----------|-------------|
| [22-suspense.md](spec/22-suspense.md) | Async data loading with Suspense boundaries |
| [23-portals.md](spec/23-portals.md) | Out-of-tree rendering for modals, tooltips, dropdowns |
| [24-head-management.md](spec/24-head-management.md) | Dynamic document head (Title, Meta, Link) |
| [25-transitions.md](spec/25-transitions.md) | CSS transitions, animations, View Transitions API |
| [26-slots.md](spec/26-slots.md) | Named slots for component composition |

### Platform Integration

| Document | Description |
|-----------|-------------|
| [27-spin-sdk.md](spec/27-spin-sdk.md) | Native Zig SDK for Fermyon Spin |

### Examples and Testing

| Document | Description |
|-----------|-------------|
| [16-examples.md](spec/16-examples.md) | Complete application examples |
| [17-tests.md](spec/17-tests.md) | Test suite specification |

---

## KEY CHANGES SINCE V5

### 1. No Global State

**Before (v5):**
```zig
var global_render_context: ?*RenderContext = null;
var global_theme: Theme = .{};
```

**After (v6):**
```zig
pub const AppContext = struct {
    allocator: std.mem.Allocator,
    frame_arena: *std.heap.ArenaAllocator,
    theme: *const Theme,
    router: *Router,
    effects: *EffectRunner,
};
```

### 2. Complete Virtual DOM

**Before (v5):**
```zig
pub fn map(...) Element(ParentMsg) {
    // This is a simplified placeholder
    return .none;
}
```

**After (v6):**
- Complete O(n) diffing algorithm
- Support for keyed children
- Component lifecycle hooks
- Efficient patch generation

### 3. Type-Safe Routing

**Before (v5):** Not implemented

**After (v6):**
```zig
const routes = Router.define(.{
    .home = Route.static("/"),
    .user = Route.param("/user/:id", struct { id: u32 }),
    .post = Route.param("/blog/:slug", struct { slug: []const u8 }),
});

// Type-safe navigation
router.navigate(routes.user, .{ .id = 42 });
```

### 4. Real CSS Extraction

**Before (v5):** Only generated reset and theme variables

**After (v6):**
- Static analysis of component files
- Extraction of all `styles.create()` calls
- Atomic class deduplication
- Source maps for debugging

### 5. Effects System

**Before (v5):** Not implemented

**After (v6):**
```zig
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .fetch_users => Effect.http(.{
            .url = "/api/users",
            .on_success = .users_loaded,
            .on_error = .fetch_failed,
        }),
        .start_timer => Effect.interval(1000, .tick),
        _ => Effect.none,
    };
}
```

### 6. Enhanced JS Runtime

**Before (v5):** Basic DOM operations

**After (v6):**
- Update batching with requestAnimationFrame
- Error boundaries and recovery
- Memory leak prevention
- Event delegation
- Promise-based async operations

---

## PERFORMANCE GOALS

| Metric | Target | Notes |
|---------|----------|-------|
| Initial bundle (WASM) | < 80KB gzipped | Core framework only |
| CSS output | < 5KB gzipped | Atomic deduplication |
| First contentful paint | < 100ms | After WASM load |
| Update latency | < 16ms | 60fps target |
| Memory per component | < 1KB | Excluding user data |

---

## COMPATIBILITY

### Browser Support
- Chrome 89+
- Firefox 89+
- Safari 15+
- Edge 89+

### Zig Version
- Minimum: 0.14.0
- Recommended: 0.15.x

### Build Requirements
- Zig compiler
- Fermyon Spin CLI (for deployment)

---

## DEVELOPMENT PHASES

Development follows a prototype-driven approach to validate technical assumptions:

```
Phase 0: Core Infrastructure
    │
    ▼
Phase 1: Prototype A (SPA) ──────── Enterprise applications
    │
    ├──────────────────────┐
    ▼                      ▼
Phase 2: Prototype B    Phase 3: Prototype C
(SSR)                   (SSG)
    │                      │
Complex pages &         Simple static
full applications       websites only
    │                      │
    └──────────┬───────────┘
               ▼
        Production Ready
```

| Phase | Focus | Target Use Case |
|-------|-------|-----------------|
| **Phase 0** | Core infrastructure | Build tooling validation |
| **Phase 1 (A)** | SPA | Enterprise apps, dashboards, complex interactivity |
| **Phase 2 (B)** | SSR | Content sites with SEO, pages that scale to apps |
| **Phase 3 (C)** | SSG | Simple static sites, docs, blogs |

**Key insight:** All modes converge to SPA behavior after initial load. SSR and SSG provide better first-paint and SEO, then transfer state to WASM which takes full control.

See [00-overview.md](spec/00-overview.md) for detailed phase descriptions.

---

## FILE STRUCTURE

```
zui/
├── build.zig
├── build.zig.zon
├── src/
│   ├── main.zig
│   ├── core/
│   │   ├── context.zig
│   │   ├── memory.zig
│   │   └── string_utils.zig
│   ├── vdom/
│   │   ├── element.zig
│   │   ├── diff.zig
│   │   ├── patch.zig
│   │   └── reconciler.zig
│   ├── css/
│   │   ├── types.zig
│   │   ├── rules.zig
│   │   ├── builder.zig
│   │   ├── theme.zig
│   │   ├── styles.zig
│   │   └── animations.zig
│   ├── router/
│   │   ├── router.zig
│   │   ├── route.zig
│   │   └── history.zig
│   ├── effects/
│   │   ├── effect.zig
│   │   ├── runner.zig
│   │   └── subscriptions.zig
│   ├── runtime/
│   │   ├── wasm.zig
│   │   └── app.zig
│   └── html/
│       ├── html.zig
│       └── attrs.zig
├── build/
│   ├── css_extractor.zig
│   └── analyzer.zig
├── www/
│   ├── index.html
│   ├── zui.js
│   └── styles.css
└── tests/
    └── *.zig
```

---

## QUICK START

```zig
// src/main.zig
const std = @import("std");
const zui = @import("zui");

const App = struct {
    pub const Model = struct {
        count: i32 = 0,
    };

    pub const Msg = enum {
        increment,
        decrement,
    };

    pub fn init() Model {
        return .{};
    }

    pub fn update(model: *Model, msg: Msg, ctx: *zui.AppContext) zui.Effect(Msg) {
        _ = ctx;
        switch (msg) {
            .increment => model.count += 1,
            .decrement => model.count -= 1,
        }
        return .none;
    }

    pub fn view(model: *const Model, ctx: *zui.AppContext) zui.Element(Msg) {
        const styles = zui.css.styles;

        return zui.h.div(ctx, .{
            .style = styles.create(.{
                .display = .flex,
                .flex_direction = .column,
                .align_items = .center,
                .gap = .{ .rem = 1 },
            }),
        }, &.{
            zui.h.h1(ctx, .{}, &.{
                zui.h.text(ctx, "Counter"),
            }),
            zui.h.p(ctx, .{
                .style = styles.create(.{
                    .font_size = .{ .rem = 2 },
                }),
            }, &.{
                zui.h.textFmt(ctx, "{d}", .{model.count}),
            }),
            zui.h.div(ctx, .{
                .style = styles.create(styles.container.flex()),
            }, &.{
                zui.h.button(ctx, .{
                    .onClick = .decrement,
                    .style = styles.button.secondary(),
                }, &.{
                    zui.h.text(ctx, "-"),
                }),
                zui.h.button(ctx, .{
                    .onClick = .increment,
                    .style = styles.button.primary(),
                }, &.{
                    zui.h.text(ctx, "+"),
                }),
            }),
        });
    }
};

pub fn main() void {
    zui.app.run(App, .{});
}
```

---

## LICENSE

MIT License - See LICENSE file for details.

---

## CHANGELOG

### v6.0.0
- Complete Virtual DOM implementation
- Context-based architecture (no globals)
- Type-safe routing system
- Effects system for async operations
- Real CSS extraction via static analysis
- Enhanced JS runtime with batching
- Fermyon Spin platform integration
- SPA, SSR, and SSG rendering modes

### v5.0.0
- Initial ZSS (Zig Style Sheets) implementation
- Basic WASM runtime
- Atomic CSS generation
