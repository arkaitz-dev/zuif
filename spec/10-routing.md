# 10 - ROUTING

**Document:** Type-Safe Routing System
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

The ZUI routing system provides type-safe navigation with route and parameter validation at compile time.

---

## ARCHITECTURE

### Route Definition

```zig
// src/router/route.zig
const std = @import("std");

pub const Route = struct {
    path: []const u8,
    params_type: ?type,

    /// Static route without parameters
    pub fn static(path: []const u8) Route {
        return .{
            .path = path,
            .params_type = null,
        };
    }

    /// Parameterized route with type-safe params
    pub fn param(path: []const u8, comptime ParamsType: type) Route {
        return .{
            .path = path,
            .params_type = ParamsType,
        };
    }

    /// Build path with parameters
    pub fn build(self: Route, params: anytype) ![]const u8 {
        if (self.params_type == null) {
            return self.path;
        }

        // Replace :param with actual values
        var result = std.ArrayList(u8).init(std.heap.page_allocator);
        var i: usize = 0;

        while (i < self.path.len) {
            if (self.path[i] == ':') {
                // Extract parameter name
                i += 1;
                const start = i;
                while (i < self.path.len and self.path[i] != '/') : (i += 1) {}
                const param_name = self.path[start..i];

                // Get value from params struct
                inline for (std.meta.fields(@TypeOf(params))) |field| {
                    if (std.mem.eql(u8, field.name, param_name)) {
                        const value = @field(params, field.name);
                        try result.writer().print("{any}", .{value});
                    }
                }
            } else {
                try result.append(self.path[i]);
                i += 1;
            }
        }

        return result.toOwnedSlice();
    }
};
```

### Router Implementation

```zig
// src/router/router.zig
const std = @import("std");
const Route = @import("route.zig").Route;

pub const Router = struct {
    allocator: std.mem.Allocator,
    current_path: []const u8,
    routes: std.StringHashMap(Route),
    listeners: std.ArrayList(*const fn ([]const u8) void),

    pub fn init(allocator: std.mem.Allocator) !Router {
        return .{
            .allocator = allocator,
            .current_path = "/",
            .routes = std.StringHashMap(Route).init(allocator),
            .listeners = std.ArrayList(*const fn ([]const u8) void).init(allocator),
        };
    }

    pub fn deinit(self: *Router) void {
        self.routes.deinit();
        self.listeners.deinit();
    }

    /// Define routes at compile time
    pub fn define(comptime routes: anytype) @TypeOf(routes) {
        return routes;
    }

    /// Register a route
    pub fn register(self: *Router, name: []const u8, route: Route) !void {
        try self.routes.put(name, route);
    }

    /// Navigate to a route with type-safe params
    pub fn navigate(self: *Router, comptime route: Route, params: anytype) !void {
        const path = try route.build(params);
        try self.push(path);
    }

    /// Push a new path
    pub fn push(self: *Router, path: []const u8) !void {
        self.current_path = path;

        // Call JS to update browser history
        js_pushState(path.ptr, path.len);

        // Notify listeners
        for (self.listeners.items) |listener| {
            listener(path);
        }
    }

    /// Go back in history
    pub fn back(self: *Router) void {
        js_goBack();
    }

    /// Go forward in history
    pub fn forward(self: *Router) void {
        js_goForward();
    }

    /// Get current path
    pub fn currentPath(self: *const Router) []const u8 {
        return self.current_path;
    }

    /// Add route change listener
    pub fn addListener(self: *Router, listener: *const fn ([]const u8) void) !void {
        try self.listeners.append(listener);
    }
};

// WASM imports
extern "zui" fn js_pushState(path_ptr: [*]const u8, path_len: usize) void;
extern "zui" fn js_replaceState(path_ptr: [*]const u8, path_len: usize) void;
extern "zui" fn js_goBack() void;
extern "zui" fn js_goForward() void;
```

---

## USAGE

### Route Definition

```zig
const routes = Router.define(.{
    .home = Route.static("/"),
    .about = Route.static("/about"),
    .user = Route.param("/user/:id", struct { id: u32 }),
    .post = Route.param("/blog/:year/:slug", struct {
        year: u32,
        slug: []const u8
    }),
});
```

### Type-Safe Navigation

```zig
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .go_home => {
            try ctx.router.navigate(routes.home, .{});
            return Effect.none;
        },

        .view_user => |user_id| {
            try ctx.router.navigate(routes.user, .{ .id = user_id });
            return Effect.none;
        },

        .view_post => |data| {
            try ctx.router.navigate(routes.post, .{
                .year = data.year,
                .slug = data.slug,
            });
            return Effect.none;
        },
    };
}
```

### Route Matching

```zig
pub fn view(model: *const Model, ctx: *AppContext) Element(Msg) {
    const path = ctx.router.currentPath();

    if (matchRoute(path, routes.home)) {
        return HomePage(ctx, model);
    } else if (matchRoute(path, routes.user)) |params| {
        return UserPage(ctx, model, params.id);
    } else if (matchRoute(path, routes.post)) |params| {
        return PostPage(ctx, model, params.year, params.slug);
    } else {
        return NotFoundPage(ctx);
    }
}

fn matchRoute(path: []const u8, comptime route: Route) ?@TypeOf(route.params_type) {
    // Pattern matching implementation
    // Returns params if match, null otherwise
}
```

---

## BROWSER INTEGRATION

### JavaScript Runtime

```javascript
// www/zui.js - Router support
const ZuiRouter = {
    init(wasmExports) {
        window.addEventListener('popstate', (event) => {
            const path = window.location.pathname;
            wasmExports.router_onPopState(
                strToPtr(path),
                path.length
            );
        });
    },

    pushState(pathPtr, pathLen) {
        const path = ptrToStr(pathPtr, pathLen);
        window.history.pushState({}, '', path);
    },

    replaceState(pathPtr, pathLen) {
        const path = ptrToStr(pathPtr, pathLen);
        window.history.replaceState({}, '', path);
    },

    goBack() {
        window.history.back();
    },

    goForward() {
        window.history.forward();
    }
};
```

---

## LINK COMPONENTS

```zig
/// Type-safe link component
pub fn Link(
    ctx: *AppContext,
    comptime Msg: type,
    comptime route: Route,
    params: anytype,
    children: []Element(Msg),
) Element(Msg) {
    const path = route.build(params) catch "/";

    return h.a(ctx, Msg, .{
        .href = path,
        .onClick = .{ .navigate = .{ .route = route, .params = params } },
    }, children);
}

// Usage
Link(ctx, Msg, routes.user, .{ .id = 42 }, &.{
    h.text(ctx, Msg, "View User Profile"),
});
```

---

## CONCLUSION

The type-safe routing system enables safe navigation and parameter validation at compile time.

**Links:**
- [← Previous: CSS Theme](09-css-theme.md)
- [→ Next: Effects System](11-effects-system.md)
- [↑ Back to index](../README.md)
