# 18 - FERMYON SPIN DEPLOYMENT

**Document:** Fermyon Spin Integration and Rendering Modes
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

ZUI is designed to run on the Fermyon Spin platform, providing a unified deployment target for all rendering modes. This document specifies how ZUI applications are built, configured, and deployed to Fermyon Spin.

---

## FERMYON SPIN PLATFORM

### What is Fermyon Spin?

Fermyon Spin is a framework for building and running fast, secure, and composable cloud microservices with WebAssembly. It provides:

- **Edge-ready execution:** Sub-millisecond cold starts
- **WebAssembly isolation:** Secure sandboxed execution
- **Static file serving:** Built-in file server component
- **HTTP triggers:** Route-based request handling
- **Key-value storage:** Persistent state management
- **Outbound HTTP:** External API communication

### Why Fermyon Spin for ZUI?

1. **Unified platform:** Single deployment target for SPA, SSR, and SSG
2. **WebAssembly native:** Both client and server code compile to WASM
3. **Zig compatibility:** wasm32-wasi target works with Zig
4. **Performance:** Near-instant startup times
5. **Simplicity:** Single binary deployment model

---

## STATE TRANSFER (NOT HYDRATION)

ZUI uses **State Transfer** instead of traditional hydration. This is a key architectural decision:

### Traditional Hydration (React, Vue, etc.)
```
Server: Render HTML → Send to client
Client: Parse HTML → Execute JS → Reconcile DOM → Attach events
Problem: Client re-executes rendering logic, must match server output exactly
```

### ZUI State Transfer
```
Server: Render HTML + Serialize Model → Send to client
Client: Load WASM → Parse Model JSON → Take full DOM control → Render fresh
Advantage: No reconciliation, simpler code, guaranteed consistency
```

### Why State Transfer Works for ZUI

1. **Elm Architecture**: The `Model` is the single source of truth
   - Model is explicit and serializable (no closures, no hidden state)
   - View is a pure function: `view(model) → VDOM`
   - Same model always produces same view

2. **Small WASM Bundle**: Target < 80KB gzipped
   - Fast to load and initialize
   - No penalty for "re-rendering" client-side

3. **Simplicity**: No complex reconciliation logic
   - Server renders HTML for fast first paint + SEO
   - Client WASM takes complete control when ready
   - No hydration mismatches possible

### State Transfer Flow

```
┌─────────────────────────────────────────────────────────────┐
│ SERVER (SSR) or BUILD (SSG)                                 │
│                                                             │
│   Model { count: 5, user: "Alice" }                        │
│         │                                                   │
│         ├──▶ view(model) ──▶ HTML (for first paint)        │
│         │                                                   │
│         └──▶ JSON.stringify(model) ──▶ <script> tag        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ CLIENT (Browser)                                            │
│                                                             │
│   1. Display pre-rendered HTML immediately (functional)    │
│   2. Load app.wasm in parallel                             │
│   3. Parse JSON state from <script> tag                    │
│   4. Initialize Model in WASM memory with parsed state     │
│   5. Adopt existing DOM (attach event handlers)            │
│   6. WASM takes control - NO re-render needed              │
│   7. Continue as SPA (future updates use VDOM diff)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Critical insight:** The WASM does NOT re-render the initial view. The pre-rendered HTML is already correct (same Model → same View). WASM only needs to:
- Load the state into memory
- Attach event handlers to existing DOM elements
- Start listening for user interactions

---

## RENDERING MODES AND USE CASES

### Intended Use by Mode

| Mode | Intended For | NOT Intended For |
|------|--------------|------------------|
| **SPA** | Enterprise applications, dashboards, admin panels, complex interactive tools | Simple content sites, SEO-critical pages |
| **SSR** | Content sites needing SEO, e-commerce, pages that may grow into full apps | Pure static content, documentation |
| **SSG** | Documentation, blogs, landing pages, marketing sites | Enterprise applications, dynamic content, complex interactivity |

### The Convergence Principle

All three modes **converge to SPA behavior** after initial load:

```
SPA:  [Shell] ──── WASM loads ──── [Interactive SPA]
SSR:  [Full HTML] ─ WASM loads ─ adopt() ─ [Interactive SPA]
SSG:  [Full HTML] ─ WASM loads ─ adopt() ─ [Interactive SPA]
                                              ↑
                              All modes end here
```

The difference is only in **how the first page is delivered**:
- **SPA:** Minimal shell, WASM builds everything
- **SSR:** Server builds HTML, WASM adopts it
- **SSG:** Build-time HTML, WASM adopts it

### Mode Comparison

| Feature | SPA | SSR | SSG |
|---------|-----|-----|-----|
| Initial HTML | Minimal shell | Full page | Full page |
| SEO friendly | Limited | Excellent | Excellent |
| Time to First Byte | Fast | Medium | Fast |
| Time to Interactive | After WASM load | After WASM load | After WASM load |
| Server required | No (static) | Yes (WASI) | No (static) |
| Dynamic content | Client-side | Server + Client | Build-time + Client |
| Personalization | Client-side | Server-side | Limited |
| State transfer | N/A | Server → Client | Build → Client |
| After first load | SPA | SPA | SPA |

### 1. SPA Mode (Single Page Application)

```
Request Flow:
┌────────┐     ┌─────────────────┐     ┌────────────┐
│ Browser│────▶│  Spin Static    │────▶│ index.html │
│        │     │  File Server    │     │ app.wasm   │
└────────┘     └─────────────────┘     │ styles.css │
                                       └────────────┘
                                              │
                                              ▼
                                       ┌────────────┐
                                       │  Browser   │
                                       │  Renders   │
                                       │  via WASM  │
                                       └────────────┘
```

**Use cases:**
- Dashboards and admin panels
- Interactive applications
- Internal tools
- Applications behind authentication

**Configuration:**
```toml
# spin.toml for SPA
spin_manifest_version = 2

[application]
name = "zui-spa"
version = "1.0.0"

[[trigger.http]]
route = "/..."
component = "static"

[component.static]
source = { url = "https://github.com/fermyon/spin-fileserver/releases/download/v0.3.0/spin_static_fs.wasm", digest = "sha256:..." }
files = [{ source = "www/", destination = "/" }]

[component.static.environment]
FALLBACK_PATH = "index.html"
```

### 2. SSR Mode (Server-Side Rendering)

```
Request Flow:
┌────────┐     ┌─────────────────┐     ┌────────────────────┐
│ Browser│────▶│   Spin HTTP     │────▶│   server.wasm      │
│        │     │   Trigger       │     │   (wasm32-wasi)    │
└────────┘     └─────────────────┘     └────────────────────┘
     ▲                                          │
     │                                          ▼
     │                               ┌────────────────────┐
     │                               │  Generate HTML +   │
     │                               │  Serialize Model   │
     │                               └────────────────────┘
     │                                          │
     └──────────────────────────────────────────┘
              HTML + State Response
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                      Browser                             │
│  1. Render HTML immediately (fully functional)          │
│  2. Load app.wasm in parallel                           │
│  3. WASM reads serialized state                         │
│  4. WASM takes control → operates as SPA                │
└─────────────────────────────────────────────────────────┘
```

**Key concept: State Transfer (not Hydration)**

Unlike traditional hydration, ZUI uses state transfer:
- Server serializes the `Model` (Elm Architecture state) into HTML
- Client WASM reads this state and initializes with it
- No reconciliation needed - WASM takes full control
- Works because `view` is a pure function of `Model`

**Use cases:**
- Content sites requiring SEO
- E-commerce product pages
- Social media previews
- Personalized content

**Configuration:**
```toml
# spin.toml for SSR
spin_manifest_version = 2

[application]
name = "zui-ssr"
version = "1.0.0"

# API routes
[[trigger.http]]
route = "/api/..."
component = "api"

# Static assets (JS, CSS, images)
[[trigger.http]]
route = "/static/..."
component = "static"

# SSR for all other routes
[[trigger.http]]
route = "/..."
component = "ssr"

[component.api]
source = "target/wasm32-wasi/release/server.wasm"
allowed_outbound_hosts = ["https://*"]
key_value_stores = ["default"]

[component.ssr]
source = "target/wasm32-wasi/release/server.wasm"
allowed_outbound_hosts = ["https://*"]

[component.static]
source = { url = "https://github.com/fermyon/spin-fileserver/releases/download/v0.3.0/spin_static_fs.wasm", digest = "sha256:..." }
files = [{ source = "www/", destination = "/" }]
```

### 3. SSG Mode (Static Site Generation)

```
Build Time:
┌────────────┐     ┌─────────────────────┐     ┌────────────┐
│ ssg.zig    │────▶│  Generate HTML +    │────▶│ *.html     │
│ (host)     │     │  Serialize State    │     │ styles.css │
└────────────┘     │  for all routes     │     │ app.wasm   │
                   └─────────────────────┘     └────────────┘

Runtime (First Request):
┌────────┐     ┌─────────────────┐     ┌──────────────────┐
│ Browser│────▶│  Spin Static    │────▶│ page.html +      │
│        │     │  File Server    │     │ serialized state │
└────────┘     └─────────────────┘     │ + app.wasm       │
                                       └──────────────────┘
                                              │
                                              ▼
                                ┌─────────────────────────┐
                                │  WASM loads, reads      │
                                │  state, takes control   │
                                │  → operates as SPA      │
                                └─────────────────────────┘

Subsequent Navigation (client-side):
┌────────────────────────────────────────────────────────┐
│  All navigation handled by WASM client-side           │
│  No additional server requests for page content       │
└────────────────────────────────────────────────────────┘
```

**Key difference from SPA:**
- First paint is instant (pre-rendered HTML)
- SEO-friendly (content in initial HTML)
- After WASM loads, behaves identically to SPA

**Use cases:**
- Documentation sites
- Blogs and marketing pages
- Landing pages
- Sites with infrequent updates

**Configuration:**
```toml
# spin.toml for SSG
spin_manifest_version = 2

[application]
name = "zui-ssg"
version = "1.0.0"

[[trigger.http]]
route = "/..."
component = "static"

[component.static]
source = { url = "https://github.com/fermyon/spin-fileserver/releases/download/v0.3.0/spin_static_fs.wasm", digest = "sha256:..." }
files = [{ source = "www/", destination = "/" }]
```

---

## SERVER-SIDE IMPLEMENTATION

### SSR Server Component

```zig
// src/server.zig
const std = @import("std");
const zui = @import("zui");
const spin = @import("spin");

// Spin HTTP handler
export fn handle_request(req: *spin.Request) spin.Response {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const allocator = arena.allocator();

    // Parse route
    const path = req.uri();
    const route = zui.Router.match(path) catch {
        return spin.Response.notFound();
    };

    // Initialize app context for SSR
    var ctx = zui.ServerContext.init(allocator, .{
        .request = req,
        .route = route,
    });

    // Render to HTML string
    const html = zui.renderToString(App, &ctx) catch |err| {
        std.log.err("SSR render failed: {}", .{err});
        return spin.Response.internalError();
    };

    // Return HTML response
    return spin.Response{
        .status = 200,
        .headers = &.{
            .{ "Content-Type", "text/html; charset=utf-8" },
        },
        .body = html,
    };
}

// Application definition (shared with client)
const App = @import("app.zig").App;
```

### HTML Generation with State Serialization

```zig
// src/ssr/render.zig
const std = @import("std");
const zui = @import("zui");

pub fn renderToString(
    comptime AppType: type,
    ctx: *zui.ServerContext,
) ![]const u8 {
    const allocator = ctx.allocator;

    // Initialize model (may include data from request/database)
    var model = AppType.init();

    // If SSR with data fetching, populate model
    if (ctx.mode == .ssr) {
        try AppType.fetchData(&model, ctx);
    }

    // Generate VDOM
    const vdom = AppType.view(&model, ctx);

    // Serialize model to JSON for state transfer
    const model_json = try std.json.stringifyAlloc(allocator, model, .{});

    // Serialize to HTML
    var html = std.ArrayList(u8).init(allocator);
    const writer = html.writer();

    // Write document shell
    try writer.writeAll(
        \\<!DOCTYPE html>
        \\<html lang="en">
        \\<head>
        \\  <meta charset="UTF-8">
        \\  <meta name="viewport" content="width=device-width, initial-scale=1.0">
        \\  <link rel="stylesheet" href="/static/styles.css">
        \\</head>
        \\<body>
        \\  <div id="app">
    );

    // Render VDOM to HTML
    try renderVdomToHtml(writer, vdom);

    // Embed serialized state for client pickup
    try writer.writeAll(
        \\  </div>
        \\  <script id="__ZUI_STATE__" type="application/json">
    );
    try writer.writeAll(model_json);
    try writer.writeAll(
        \\  </script>
        \\  <script type="module" src="/static/zui.js"></script>
        \\  <script type="module">
        \\    import { initFromState } from '/static/zui.js';
        \\    initFromState('/static/app.wasm');
        \\  </script>
        \\</body>
        \\</html>
    );

    return html.toOwnedSlice();
}

fn renderVdomToHtml(writer: anytype, element: zui.Element) !void {
    switch (element) {
        .text => |text| try writer.writeAll(escapeHtml(text)),
        .element => |el| {
            try writer.print("<{s}", .{el.tag});

            // Write attributes
            for (el.attrs) |attr| {
                try writer.print(" {s}=\"{s}\"", .{ attr.key, escapeAttr(attr.value) });
            }

            if (el.children.len == 0 and isVoidElement(el.tag)) {
                try writer.writeAll(" />");
            } else {
                try writer.writeAll(">");
                for (el.children) |child| {
                    try renderVdomToHtml(writer, child);
                }
                try writer.print("</{s}>", .{el.tag});
            }
        },
        .none => {},
    }
}
```

### Client State Transfer

```zig
// src/client.zig - Client entry point with state transfer
const std = @import("std");
const zui = @import("zui");

/// Called by JS runtime after WASM loads
export fn initWithState(state_ptr: [*]const u8, state_len: usize) void {
    const state_json = state_ptr[0..state_len];

    // Parse serialized state from server
    var model = std.json.parse(
        App.Model,
        state_json,
        .{ .allocator = zui.allocator },
    ) catch {
        // Fallback: initialize fresh if state parsing fails
        model = App.init();
    };

    // Initialize app context
    var ctx = zui.AppContext.init(zui.allocator);

    // Get root element (already contains pre-rendered HTML)
    const root = zui.dom.getElementById("app") orelse return;

    // Generate VDOM to know where to attach event handlers
    // This does NOT modify the DOM - it's just for event binding
    const vdom = App.view(&model, &ctx);

    // Adopt existing DOM: walk VDOM and attach event handlers
    // to corresponding DOM elements (no DOM mutations)
    zui.dom.adopt(root, vdom);

    // Store current VDOM for future diffing
    ctx.setCurrentVdom(vdom);

    // App is now fully interactive as SPA
    // Future updates will use normal VDOM diff + patch
    zui.runtime.start(&model, &ctx, App.update, App.view);
}

const App = @import("app.zig").App;
```

---

## TECHNICAL DEEP DIVE: THE `adopt()` FUNCTION

The `adopt()` function is the critical piece that enables State Transfer. It allows the client WASM to take control of server-rendered DOM without re-rendering.

### The Core Problem

```
Server generates:  <button class="btn">Count: 5</button>
Client has:        VDOM node { tag: "button", events: [onClick → increment], ... }

Question: How does the client know THIS DOM button corresponds to THIS VDOM node?
```

### Strategy: Parallel Tree Walking

Since the server and client use **identical `view()` functions** with **identical `Model` state**, the DOM and VDOM trees have **identical structure**. We can walk both trees in parallel:

```zig
// src/runtime/adopt.zig
const std = @import("std");
const zui = @import("zui");

pub fn adopt(dom_node: DomRef, vdom: Element, ctx: *AdoptContext) !void {
    switch (vdom) {
        .text => {
            // Text nodes have no events, just store reference
            ctx.registerNode(dom_node);
        },
        .element => |el| {
            // Store reference for future updates
            const node_id = ctx.registerNode(dom_node);

            // Attach event handlers from VDOM to DOM element
            for (el.events) |event| {
                try attachEvent(node_id, event.type, event.msg_id);
            }

            // Recursively adopt children (parallel walk)
            var dom_children = try getDomChildren(dom_node);
            for (el.children, 0..) |vdom_child, i| {
                if (i < dom_children.len) {
                    try adopt(dom_children[i], vdom_child, ctx);
                }
            }
        },
        .none => {},
    }
}

fn attachEvent(node_id: u32, event_type: EventType, msg_id: u32) !void {
    // Call JS to attach event listener
    js_addEventListener(node_id, @tagName(event_type), msg_id);
}

// Imported from JS runtime
extern "zui" fn js_addEventListener(node_id: u32, event_ptr: [*]const u8, event_len: usize, msg_id: u32) void;
extern "zui" fn js_getChildren(node_id: u32) u32; // Returns child count
extern "zui" fn js_getChild(parent_id: u32, index: u32) u32; // Returns child node_id
```

### Why Parallel Walking Works

1. **Same code path:** Server and client execute identical `view(model)` function
2. **Same input:** Model state is transferred via JSON, producing identical VDOM
3. **Deterministic rendering:** Pure function guarantees same output for same input
4. **Structural equivalence:** DOM structure matches VDOM structure exactly

```
Server VDOM:                    Server DOM:
div                             <div>
├── h1                            <h1>Counter</h1>
├── p                             <p>Count: 5</p>
└── button [onClick]              <button>Increment</button>
                                </div>

Client VDOM:                    Walking:
div ─────────────────────────── <div> ✓
├── h1 ──────────────────────── <h1> ✓
├── p ───────────────────────── <p> ✓
└── button [onClick] ─────────── <button> ✓ (attach onClick)
```

### Edge Cases and Mitigations

#### 1. Conditional Rendering
```zig
// If this condition differs between server/client, trees won't match
if (model.show_banner) {
    return h.div(...);
}
```
**Mitigation:** Model is transferred, so conditions evaluate identically.

#### 2. Dynamic Lists
```zig
// List items must be in same order
for (model.items) |item| {
    return h.li(.{ .key = item.id }, ...);
}
```
**Mitigation:** Keys ensure consistent ordering; Model transfer preserves order.

#### 3. Time-dependent Rendering
```zig
// ❌ BAD: Different time on server vs client
const now = std.time.timestamp();
return h.span(.{}, formatTime(now));
```
**Mitigation:** Time-dependent data should be in Model, transferred from server.

### Failure Modes and Recovery

If trees don't match (bug or edge case):

```zig
pub fn adopt(dom_node: DomRef, vdom: Element, ctx: *AdoptContext) !void {
    // Validate structure matches
    if (!structureMatches(dom_node, vdom)) {
        // Log warning in development
        if (comptime std.debug.runtime_safety) {
            std.log.warn("adopt() structure mismatch, falling back to re-render", .{});
        }

        // Fallback: clear and re-render (loses SSR benefit but works)
        js_clearElement(ctx.root_id);
        return render(ctx.root_id, vdom, ctx);
    }

    // ... normal adopt logic
}

fn structureMatches(dom: DomRef, vdom: Element) bool {
    return switch (vdom) {
        .text => dom.nodeType == .TEXT_NODE,
        .element => |el| dom.nodeType == .ELEMENT_NODE and
                        std.mem.eql(u8, dom.tagName, el.tag),
        .none => true,
    };
}
```

### Performance Characteristics

| Operation | Cost |
|-----------|------|
| Tree walking | O(n) where n = number of nodes |
| Event attachment | O(e) where e = number of events |
| DOM queries | Minimal (children access only) |
| DOM mutations | **Zero** (adopt never modifies DOM) |

**Comparison with Hydration (React-style):**

| Aspect | ZUI adopt() | React hydration |
|--------|-------------|-----------------|
| DOM reads | Children only | Full tree comparison |
| DOM writes | None | Potentially many (mismatches) |
| JS execution | Parse state | Re-execute components |
| Complexity | Simple tree walk | Complex reconciliation |

---

## WHY NOT TRADITIONAL HYDRATION?

Traditional hydration (React, Vue, etc.) has inherent problems:

### 1. Hydration Mismatches
```javascript
// React: Server and client can produce different output
function Component() {
    // This differs between server/client!
    const isClient = typeof window !== 'undefined';
    return <div>{isClient ? 'Client' : 'Server'}</div>;
}
// Result: Hydration mismatch warning, potential UI bugs
```

### 2. Double Rendering Work
```
Server: model → VDOM → HTML string
Client: HTML → parse → model → VDOM → compare → attach events
               ↑ wasted work ↑
```

### 3. Flash of Incorrect Content
If hydration finds mismatches, React may re-render, causing visible flashes.

### ZUI's State Transfer Avoids All This

```
Server: model → VDOM → HTML string
        model → JSON ──────────────┐
                                   │
Client: JSON → model              ←┘
        model → VDOM (for events only)
        adopt(DOM, VDOM) → attach events
        Done. No comparison, no re-render.
```

**Key function: `zui.dom.adopt()`**

Unlike `render()` which creates DOM nodes, `adopt()` walks the existing DOM in parallel with the VDOM and:
- Attaches event handlers to existing elements
- Stores element references for future updates
- Does NOT modify the DOM structure or content

```javascript
// www/zui.js - State transfer initialization
export async function initFromState(wasmPath) {
    // Read serialized state from embedded script
    const stateEl = document.getElementById('__ZUI_STATE__');
    const stateJson = stateEl ? stateEl.textContent : '{}';

    // Load WASM
    const response = await fetch(wasmPath);
    const bytes = await response.arrayBuffer();
    const { instance } = await WebAssembly.instantiate(bytes, imports);

    // Transfer state to WASM
    const encoder = new TextEncoder();
    const stateBytes = encoder.encode(stateJson);
    const ptr = instance.exports.alloc(stateBytes.length);
    new Uint8Array(instance.exports.memory.buffer, ptr, stateBytes.length)
        .set(stateBytes);

    // Initialize WASM with transferred state
    instance.exports.initWithState(ptr, stateBytes.length);

    // Remove state script (no longer needed)
    stateEl?.remove();
}
```

---

## SSG IMPLEMENTATION

### Static Site Generator

```zig
// src/ssg.zig
const std = @import("std");
const zui = @import("zui");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    const output_dir = args[1];

    // Get all routes from app
    const routes = App.routes;

    // Generate HTML for each route
    for (routes) |route| {
        const html = try generatePage(allocator, route);
        defer allocator.free(html);

        const path = try std.fmt.allocPrint(
            allocator,
            "{s}/{s}.html",
            .{ output_dir, route.path },
        );
        defer allocator.free(path);

        try std.fs.cwd().writeFile(path, html);

        std.debug.print("Generated: {s}\n", .{path});
    }
}

fn generatePage(allocator: std.mem.Allocator, route: zui.Route) ![]const u8 {
    var ctx = zui.ServerContext.init(allocator, .{
        .route = route,
        .mode = .ssg,
    });

    return try zui.renderToString(App, &ctx);
}

const App = @import("app.zig").App;
```

---

## API ROUTES

### Defining API Endpoints

```zig
// src/api.zig
const std = @import("std");
const spin = @import("spin");

pub const routes = .{
    .{ "GET", "/api/users", handleGetUsers },
    .{ "POST", "/api/users", handleCreateUser },
    .{ "GET", "/api/users/:id", handleGetUser },
};

fn handleGetUsers(req: *spin.Request) spin.Response {
    // Fetch from database or external API
    const users = fetchUsers() catch {
        return spin.Response.internalError();
    };

    return spin.Response.json(users);
}

fn handleCreateUser(req: *spin.Request) spin.Response {
    const body = req.body() orelse {
        return spin.Response.badRequest("Missing body");
    };

    const user = std.json.parse(User, body, .{}) catch {
        return spin.Response.badRequest("Invalid JSON");
    };

    // Save user
    saveUser(user) catch {
        return spin.Response.internalError();
    };

    return spin.Response{
        .status = 201,
        .body = std.json.stringify(user, .{}),
    };
}
```

---

## KEY-VALUE STORAGE

### Using Spin Key-Value Store

```zig
// src/storage.zig
const spin = @import("spin");

pub const Store = struct {
    handle: spin.KeyValue,

    pub fn open(name: []const u8) !Store {
        return .{
            .handle = try spin.KeyValue.open(name),
        };
    }

    pub fn get(self: *Store, key: []const u8) !?[]const u8 {
        return self.handle.get(key);
    }

    pub fn set(self: *Store, key: []const u8, value: []const u8) !void {
        try self.handle.set(key, value);
    }

    pub fn delete(self: *Store, key: []const u8) !void {
        try self.handle.delete(key);
    }
};

// Usage in API handler
fn handleSession(req: *spin.Request) spin.Response {
    var store = Store.open("sessions") catch {
        return spin.Response.internalError();
    };

    const session_id = req.header("Cookie") orelse {
        return spin.Response.unauthorized();
    };

    const session_data = store.get(session_id) catch {
        return spin.Response.internalError();
    };

    // ...
}
```

---

## DEPLOYMENT

### Local Development

```bash
# Install Spin CLI
curl -fsSL https://developer.fermyon.com/downloads/install.sh | bash

# Build ZUI application
zig build -Dmode=ssr

# Generate spin.toml
zig build spin -Dmode=ssr

# Run locally
spin up

# Application available at http://localhost:3000
```

### Deploy to Fermyon Cloud

```bash
# Login to Fermyon Cloud
spin cloud login

# Deploy application
spin deploy

# Application deployed to https://your-app.fermyon.app
```

### Custom Domain

```bash
# Add custom domain
spin cloud variables set CUSTOM_DOMAIN=example.com

# Configure DNS CNAME record
# example.com -> your-app.fermyon.app
```

---

## HYBRID MODE

### Mixing Rendering Strategies

```zig
// src/app.zig
pub const routes = .{
    // SSG: Static marketing pages
    Route.ssg("/", HomePage),
    Route.ssg("/about", AboutPage),
    Route.ssg("/blog/:slug", BlogPost),

    // SSR: Dynamic user content
    Route.ssr("/dashboard", DashboardPage),
    Route.ssr("/profile/:id", ProfilePage),

    // SPA: Highly interactive sections
    Route.spa("/app/*", AppShell),

    // API: Data endpoints
    Route.api("/api/...", ApiHandler),
};

// Build configuration detects route types
// and generates appropriate outputs
```

---

## PERFORMANCE CONSIDERATIONS

### Cold Start Optimization

```zig
// Minimize initialization work
pub fn init() App {
    // Lazy initialization for expensive operations
    return .{
        .cache = null,  // Initialize on first use
        .db = null,
    };
}

// Use comptime where possible
const ROUTES = comptime Router.compile(route_definitions);
```

### Response Streaming (Future)

```zig
// SSR with streaming (v7.0 planned)
pub fn streamResponse(writer: anytype) !void {
    // Send head immediately
    try writer.writeAll("<!DOCTYPE html><html><head>...</head><body>");

    // Stream body chunks as they're ready
    try writer.writeAll("<div id=\"app\">");
    try streamComponent(writer, HeaderComponent);
    try writer.flush();

    try streamComponent(writer, MainContent);
    try writer.flush();

    try writer.writeAll("</div></body></html>");
}
```

---

## CONCLUSION

Fermyon Spin provides an ideal deployment platform for ZUI applications, supporting all rendering modes with a unified architecture. The combination of Zig's performance and Spin's WebAssembly runtime enables fast, secure, and scalable web applications.

**Links:**
- [← Previous: Build System](15-build-system.md)
- [← Previous: Examples](16-examples.md)
- [↑ Back to index](../README.md)
- [Fermyon Spin Documentation](https://developer.fermyon.com/spin)
