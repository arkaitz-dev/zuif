# 00 - PROJECT OVERVIEW

**Document:** Project Overview
**Version:** 6.0.0
**Status:** Final

---

## PROJECT OBJECTIVES

ZUI is a framework for building web user interfaces compiled to WebAssembly using Zig, designed to run on the Fermyon Spin platform. The framework supports multiple rendering modes: SPA (Single Page Application), SSR (Server-Side Rendering), and SSG (Static Site Generation). The main objectives are:

### 1. Type Safety Throughout
- Compile-time route validation
- Typed CSS with validated values
- Typed application messages
- No unsafe casts in user code

### 2. Zero Runtime Overhead
- CSS extracted and optimized at build time
- Efficient Virtual DOM with O(n) complexity
- No unnecessary runtime dependencies
- Minimal and optimized WASM code

### 3. Predictable Architecture
- Elm Architecture: Model → View → Update
- Unidirectional data flow
- No global mutable state
- Explicit side effects

### 4. Excellent Development Experience
- Clear compiler error messages
- Hot reload during development
- Source maps for debugging
- Complete documentation

### 5. Flexible Rendering Modes
- **SPA (Single Page Application):** Client-side rendering with WASM
- **SSR (Server-Side Rendering):** Server-rendered HTML with hydration
- **SSG (Static Site Generation):** Pre-rendered static HTML at build time
- Seamless transition between modes

### 6. Fermyon Spin Integration
- Native deployment on Fermyon Spin platform
- Static file serving for SPA and SSG modes
- Dynamic server components for SSR and API endpoints
- Edge-ready WebAssembly execution

---

## DESIGN PRINCIPLES

### 1. Explicit over Implicit
```zig
// ✓ GOOD: Explicit context
pub fn view(model: *const Model, ctx: *AppContext) Element(Msg) {
    return h.div(ctx, .{}, &.{});
}

// ✗ BAD: Implicit global context
pub fn view(model: *const Model) Element(Msg) {
    return h.div(.{}, &.{}); // Where does the allocator come from?
}
```

### 2. Compile-Time over Runtime
```zig
// ✓ GOOD: Compile-time validation
router.navigate(routes.user, .{ .id = 42 });

// ✗ BAD: Runtime validation
router.navigate("/user/42"); // Error only at runtime if route doesn't exist
```

### 3. Composition over Inheritance
```zig
// ✓ GOOD: Component composition
pub fn UserCard(ctx: *AppContext, user: User) Element(Msg) {
    return h.div(ctx, .{}, &.{
        Avatar(ctx, user.avatar),
        UserInfo(ctx, user),
    });
}

// ✗ BAD: Base class inheritance
// There are no classes in Zig, we use composition
```

### 4. Functional over Imperative
```zig
// ✓ GOOD: Functional transformation
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    switch (msg) {
        .increment => model.count += 1,
    }
    return .none;
}

// ✗ BAD: Global mutation
// var global_count: i32 = 0;
// global_count += 1;
```

---

## RENDERING MODES

ZUI supports three rendering modes, all deployable on Fermyon Spin:

### 1. SPA (Single Page Application)
```
┌─────────────────────────────────────────────┐
│              Fermyon Spin                   │
│  ┌───────────────────────────────────────┐  │
│  │     Static File Server Component      │  │
│  │  ┌─────────────┐  ┌───────────────┐   │  │
│  │  │ index.html  │  │   app.wasm    │   │  │
│  │  │ styles.css  │  │   (client)    │   │  │
│  │  └─────────────┘  └───────────────┘   │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```
- All rendering happens in the browser
- Ideal for dashboards and interactive applications
- Fastest time-to-interactive after initial load

### 2. SSR (Server-Side Rendering)
```
┌─────────────────────────────────────────────┐
│              Fermyon Spin                   │
│  ┌───────────────────────────────────────┐  │
│  │       Dynamic WASM Component          │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │   server.wasm (wasm32-wasi)     │  │  │
│  │  │   - Route handling              │  │  │
│  │  │   - HTML generation             │  │  │
│  │  │   - API endpoints               │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
│  ┌───────────────────────────────────────┐  │
│  │     Static File Server Component      │  │
│  │  ┌─────────────┐  ┌───────────────┐   │  │
│  │  │ styles.css  │  │ app.wasm      │   │  │
│  │  │             │  │ (hydration)   │   │  │
│  │  └─────────────┘  └───────────────┘   │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```
- Initial HTML rendered on server
- Client hydrates for interactivity
- Best for SEO and initial load performance

### 3. SSG (Static Site Generation)
```
┌─────────────────────────────────────────────┐
│             Build Time                      │
│  ┌───────────────────────────────────────┐  │
│  │      SSG Generator (zig build ssg)    │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  Pre-render all routes to HTML  │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│              Fermyon Spin                   │
│  ┌───────────────────────────────────────┐  │
│  │     Static File Server Component      │  │
│  │  ┌─────────────┐  ┌───────────────┐   │  │
│  │  │ *.html      │  │ app.wasm      │   │  │
│  │  │ styles.css  │  │ (optional)    │   │  │
│  │  └─────────────┘  └───────────────┘   │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```
- All HTML generated at build time
- Optional client-side hydration for interactivity
- Best for content sites, blogs, documentation

### Hybrid Mode
ZUI supports mixing modes within a single application:
- Static pages (SSG) for content
- Dynamic pages (SSR) for personalized content
- Client-side components (SPA) for interactive widgets
- API routes for data fetching

---

## FERMYON SPIN DEPLOYMENT

### spin.toml Configuration
```toml
spin_manifest_version = 2

[application]
name = "zui-app"
version = "1.0.0"

# SSR/API Component (dynamic routes)
[[trigger.http]]
route = "/api/..."
component = "api"

[[trigger.http]]
route = "/..."
component = "ssr"

[component.api]
source = "target/wasm32-wasi/release/api.wasm"
allowed_outbound_hosts = ["https://api.example.com"]

[component.ssr]
source = "target/wasm32-wasi/release/ssr.wasm"

# Static assets (SPA/SSG)
[[trigger.http]]
route = "/static/..."
component = "static"

[component.static]
source = { url = "https://github.com/fermyon/spin-fileserver/releases/download/v0.1.0/spin_static_fs.wasm", digest = "sha256:..." }
files = [{ source = "www/", destination = "/" }]
```

### Build Targets
```bash
# SPA mode - client WASM only
zig build -Dmode=spa

# SSR mode - server + client WASM
zig build -Dmode=ssr

# SSG mode - pre-render + optional client WASM
zig build -Dmode=ssg

# Full deployment
spin build && spin deploy
```

---

## TECHNICAL CONSTRAINTS

### 1. Target Platform
- **Client Target:** `wasm32-freestanding` for browser execution
- **Server Target:** `wasm32-wasi` for Fermyon Spin execution
- **Deployment:** Fermyon Spin for both static and dynamic content
- **No libc:** Zero C dependencies
- **Size:** Target bundle < 80KB gzipped (client WASM)

### 2. Zig Version
- **Minimum:** 0.14.0
- **Recommended:** 0.15.x
- **Breaking changes:** Documented in each version

### 3. Browser Support
- **Modern:** Chrome 89+, Firefox 89+, Safari 15+, Edge 89+
- **WebAssembly:** Full support required
- **JavaScript:** ES2020+ for runtime

### 4. Memory Management
- **Arena allocator:** For frame rendering
- **GPA:** For persistent state
- **Manual cleanup:** No GC, explicit control

---

## NON-GOALS

Things that explicitly are **NOT** objectives of ZUI:

### 1. Legacy Browser Support
- No polyfills for IE11 or old browsers
- Requires native WebAssembly support
- Requires modern JavaScript (ES2020+)

### 2. Full-Stack JavaScript Interop
- No direct JavaScript framework integration (React, Vue, etc.)
- WASM-to-JS bridge is minimal and controlled
- No npm package dependencies

### 3. React/Vue Compatibility
- No interop with other frameworks
- No hybrid component system
- Standalone ecosystem

### 4. CSS-in-JS Runtime
- All CSS is extracted at build time
- No dynamic style generation
- Pre-computed atomic CSS

---

## IDEAL USE CASES

### ✓ Applications that Benefit from ZUI

1. **Dashboards and Data Applications**
   - High performance required
   - Frequent UI updates
   - Strong typing beneficial

2. **Line-of-Business Applications**
   - Complex business logic
   - Strict type validation
   - Long-term maintenance

3. **Developer Tools**
   - Need to be fast and efficient
   - Benefit from reduced size
   - Technical users

4. **Embedded Web Applications**
   - Size constraints
   - Performance needs
   - Full stack control

### ✗ Not Recommended Applications

1. **Heavy Media Applications**
   - Better to use specialized frameworks
   - ZUI not optimized for video/audio
   - Require specific APIs

3. **Rapid Prototyping**
   - Zig learning curve
   - Less mature ecosystem than React
   - Better to use prototyping tools

---

## COMPARISON WITH OTHER FRAMEWORKS

### vs React
| Aspect | ZUI | React |
|--------|-----|-------|
| Language | Zig → WASM | JavaScript/TypeScript |
| Typing | Compile-time | Runtime (TypeScript) |
| Bundle size | ~80KB | ~120KB+ |
| CSS | Static extraction | Runtime CSS-in-JS |
| Ecosystem | Small | Huge |

### vs Elm
| Aspect | ZUI | Elm |
|--------|-----|-----|
| Architecture | Elm Architecture | Elm Architecture |
| Language | Zig (systems) | Elm (pure functional) |
| Performance | Native (WASM) | JavaScript |
| Interop | Limited | Easy JavaScript |
| Curve | Steep (Zig) | Moderate |

### vs Vue
| Aspect | ZUI | Vue |
|--------|-----|-----|
| Paradigm | Functional | Reactive |
| Templates | Zig functions | HTML templates |
| Typing | Compile-time | Runtime (TypeScript) |
| Complexity | High (systems programming) | Low (progressive framework) |

---

## SUCCESS METRICS

### Performance
- [ ] First Contentful Paint < 100ms
- [ ] Time to Interactive < 200ms
- [ ] Frame budget 16ms (60fps)
- [ ] Memory per component < 1KB

### Developer Experience
- [ ] Compilation time < 5s for changes
- [ ] Clear and actionable error messages
- [ ] Complete API documentation
- [ ] Examples for common cases

### Code Quality
- [ ] 100% type-safe (no unsafe `@as()`)
- [ ] Zero memory leaks in tests
- [ ] 90%+ test coverage
- [ ] All specifications implemented

---

## DEVELOPMENT PHASES

Development follows a prototype-driven approach. Each prototype validates core technical assumptions before proceeding to the next phase.

### Phase 0: Core Infrastructure
**Objective:** Establish the foundational tooling and runtime.

- [ ] Zig → wasm32-freestanding compilation pipeline
- [ ] Basic JS runtime for WASM ↔ DOM interop
- [ ] Memory management (arena + GPA)
- [ ] Build system with `zig build`
- [ ] Development server with WebSocket
- [ ] File watcher with auto-recompilation
- [ ] Smart Reload (preserves state across reloads)
- [ ] Compile error overlay in browser

**Validation criteria:**
- "Hello World" renders in browser via WASM
- Basic event (click) triggers WASM function
- Memory allocation/deallocation works correctly
- File change triggers recompilation and browser update
- Application state preserved after reload

---

### Phase 1: Prototype A - SPA (Single Page Application)
**Objective:** Validate that complex, interactive enterprise applications are viable.

**Scope:**
- Complete Virtual DOM implementation (create, diff, patch)
- Full Elm Architecture (Model, View, Update, Effects)
- Type-safe routing with client-side navigation
- CSS system with compile-time extraction
- HTTP effects for API communication
- State management patterns

**Target applications:**
- Enterprise dashboards
- Admin panels
- Line-of-business applications
- Complex interactive tools

**Validation criteria:**
- [ ] Counter app with increment/decrement
- [ ] Todo list with add/remove/toggle
- [ ] Multi-page app with routing
- [ ] HTTP fetch and display data
- [ ] Form with validation
- [ ] Bundle size < 80KB gzipped

**Success metric:** A non-trivial application (e.g., task management system) runs entirely client-side with acceptable performance.

---

### Phase 2: Prototype B - SSR (Server-Side Rendering)
**Objective:** Validate server rendering with state transfer for content-rich pages that can scale to full applications.

**Scope:**
- Zig → wasm32-wasi compilation for Fermyon Spin
- HTML generation from VDOM on server
- Model serialization to JSON
- State Transfer mechanism (not hydration)
- `adopt()` function for DOM takeover without re-render
- API routes in same Spin application

**Target applications:**
- Content sites requiring SEO (e-commerce, blogs)
- Marketing pages with dynamic elements
- Pages that start simple but may grow into full apps
- Applications requiring fast First Contentful Paint

**Validation criteria:**
- [ ] Server generates complete HTML page
- [ ] State embedded as JSON in HTML
- [ ] Client WASM loads and parses state
- [ ] `adopt()` attaches events without DOM modification
- [ ] User interaction works immediately after adopt
- [ ] Subsequent navigation is client-side (SPA behavior)
- [ ] No visual flash or re-render on takeover

**Success metric:** A blog with comments system renders server-side, transfers state, and becomes interactive without any visible transition.

---

### Phase 3: Prototype C - SSG (Static Site Generation)
**Objective:** Validate build-time rendering for simple, static websites.

**Scope:**
- Build-time HTML generation for all routes
- State embedding in static HTML files
- Optional client WASM for interactivity
- Static deployment to Fermyon Spin

**Target applications:**
- Documentation sites
- Blogs without dynamic content
- Landing pages
- Marketing websites
- **NOT for:** Enterprise applications, complex interactivity

**Validation criteria:**
- [ ] Build generates HTML files for each route
- [ ] Pages load and display without JavaScript
- [ ] Optional: WASM enhances with interactivity
- [ ] Deploy to Spin as static files only

**Success metric:** Documentation site with 50+ pages generates in < 5 seconds and loads instantly.

---

## PROTOTYPE DEPENDENCIES

```
Phase 0: Core Infrastructure
    │
    ▼
Phase 1: Prototype A (SPA)
    │
    ├──────────────────────┐
    ▼                      ▼
Phase 2: Prototype B    Phase 3: Prototype C
(SSR)                   (SSG)
    │                      │
    └──────────┬───────────┘
               ▼
        Production Ready
```

**Key insight:** Prototype A (SPA) is the foundation. SSR and SSG are extensions that add:
- **SSR:** Server rendering + state transfer + adopt()
- **SSG:** Build-time rendering + static output

Both SSR and SSG result in SPA behavior after initial load.

---

## ROADMAP

### v0.1.0 - Core Validation
- [ ] Phase 0 complete
- [ ] Phase 1 (Prototype A) complete
- [ ] Basic documentation

### v0.2.0 - Server Rendering
- [ ] Phase 2 (Prototype B) complete
- [ ] Fermyon Spin integration validated
- [ ] State Transfer mechanism proven

### v0.3.0 - Static Generation
- [ ] Phase 3 (Prototype C) complete
- [ ] Full rendering mode coverage

### v1.0.0 - Production Ready
- [ ] All prototypes validated
- [ ] Performance optimized
- [ ] Complete documentation
- [ ] Example applications

### v1.x - Enhancements
- [ ] Animation system
- [ ] Accessibility helpers
- [ ] Developer tools
- [ ] Component library

### v2.0 - Advanced Features
- [ ] Streaming SSR responses
- [ ] Code splitting
- [ ] Partial pre-rendering

---

## CONTRIBUTION

### Principles for Contributors
1. **Safety first:** Avoid unsafe code
2. **Mandatory tests:** Complete coverage
3. **Clear documentation:** Explain the "why"
4. **Performance conscious:** Measure impact

### Review Process
1. Automated tests pass
2. Code review by 2+ maintainers
3. Performance benchmarks without regression
4. Updated documentation

---

## CONCLUSION

ZUI v6 establishes a solid foundation for building modern web applications with Zig. The focus on type safety, performance, and predictable architecture makes it ideal for applications that require robustness and long-term maintainability.

**Links:**
- [← Back to index](../README.md)
- [→ Next: Core Infrastructure](01-core-infrastructure.md)
