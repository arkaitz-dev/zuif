# 22 - SUSPENSE

**Document:** Suspense and Loading States
**Version:** 6.0.0
**Status:** Final
**Dependencies:** 03-virtual-dom.md, 11-effects-system.md

---

## OVERVIEW

Suspense provides a declarative way to handle loading states in ZUI applications. It allows components to "suspend" while waiting for asynchronous data, displaying fallback content until the data is ready.

---

## ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────┐
│                      SUSPENSE SYSTEM                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────┐ │
│  │   Suspense   │────▶│   Resource   │────▶│    View     │ │
│  │  Boundary    │     │    State     │     │  Rendering  │ │
│  └──────────────┘     └──────────────┘     └─────────────┘ │
│         │                    │                     │        │
│         ▼                    ▼                     ▼        │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────┐ │
│  │   Fallback   │     │   Loading    │     │   Content   │ │
│  │     UI       │     │   Pending    │     │   Ready     │ │
│  └──────────────┘     │   Error      │     └─────────────┘ │
│                       └──────────────┘                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## RESOURCE TYPE

### Definition

Resources represent asynchronous data that may be loading, ready, or failed.

```zig
// src/suspense/resource.zig
const std = @import("std");

/// Resource state for async data
pub fn Resource(comptime T: type) type {
    return union(enum) {
        /// Initial state - not yet requested
        idle,

        /// Loading - request in progress
        loading,

        /// Data successfully loaded
        success: T,

        /// Error occurred during loading
        failure: ResourceError,

        /// Revalidating - has data but refreshing
        revalidating: T,

        const Self = @This();

        /// Check if resource is in a loading state
        pub fn isLoading(self: Self) bool {
            return self == .loading;
        }

        /// Check if resource has data (success or revalidating)
        pub fn hasData(self: Self) bool {
            return switch (self) {
                .success, .revalidating => true,
                else => false,
            };
        }

        /// Get data if available
        pub fn getData(self: Self) ?T {
            return switch (self) {
                .success => |data| data,
                .revalidating => |data| data,
                else => null,
            };
        }

        /// Check if resource has failed
        pub fn isError(self: Self) bool {
            return self == .failure;
        }

        /// Get error if failed
        pub fn getError(self: Self) ?ResourceError {
            return switch (self) {
                .failure => |err| err,
                else => null,
            };
        }

        /// Map the success value
        pub fn map(self: Self, comptime f: fn (T) T) Self {
            return switch (self) {
                .success => |data| .{ .success = f(data) },
                .revalidating => |data| .{ .revalidating = f(data) },
                else => self,
            };
        }
    };
}

pub const ResourceError = struct {
    code: ErrorCode,
    message: []const u8,

    pub const ErrorCode = enum {
        network,
        timeout,
        not_found,
        server_error,
        parse_error,
        unknown,
    };
};
```

### Resource Creation

```zig
// src/suspense/create_resource.zig

/// Create a resource that fetches data
pub fn createResource(
    comptime T: type,
    comptime Msg: type,
    config: ResourceConfig(T, Msg),
) ResourceHandle(T, Msg) {
    return .{
        .key = config.key,
        .fetcher = config.fetcher,
        .on_success = config.on_success,
        .on_error = config.on_error,
        .options = config.options,
    };
}

pub fn ResourceConfig(comptime T: type, comptime Msg: type) type {
    return struct {
        /// Unique key for caching/deduplication
        key: []const u8,

        /// Function that returns the Effect to fetch data
        fetcher: *const fn () Effect(Msg),

        /// Message constructor for success
        on_success: *const fn (T) Msg,

        /// Message constructor for error
        on_error: *const fn (ResourceError) Msg,

        /// Optional configuration
        options: ResourceOptions = .{},
    };
}

pub const ResourceOptions = struct {
    /// Stale time in milliseconds (0 = always fresh)
    stale_time: u32 = 0,

    /// Cache time in milliseconds (how long to keep in cache)
    cache_time: u32 = 5 * 60 * 1000, // 5 minutes

    /// Retry on error
    retry: bool = true,

    /// Max retry attempts
    retry_count: u8 = 3,

    /// Retry delay in milliseconds
    retry_delay: u32 = 1000,

    /// Refetch on window focus
    refetch_on_focus: bool = false,

    /// Refetch interval (0 = disabled)
    refetch_interval: u32 = 0,
};

pub fn ResourceHandle(comptime T: type, comptime Msg: type) type {
    return struct {
        key: []const u8,
        fetcher: *const fn () Effect(Msg),
        on_success: *const fn (T) Msg,
        on_error: *const fn (ResourceError) Msg,
        options: ResourceOptions,

        const Self = @This();

        /// Trigger a fetch
        pub fn fetch(self: Self) Effect(Msg) {
            return self.fetcher();
        }

        /// Trigger a refetch (ignore cache)
        pub fn refetch(self: Self) Effect(Msg) {
            return Effect(Msg).batch(&.{
                Effect(Msg).invalidateCache(self.key),
                self.fetcher(),
            });
        }
    };
}
```

---

## SUSPENSE ELEMENT

### Core Definition

```zig
// src/suspense/suspense.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Suspense boundary element
pub fn Suspense(comptime Msg: type) type {
    return struct {
        /// Fallback content shown while loading
        fallback: vdom.Element(Msg),

        /// Main content (may contain suspended resources)
        children: []const vdom.Element(Msg),

        /// Optional error fallback
        error_fallback: ?*const fn (ResourceError) vdom.Element(Msg) = null,

        /// Minimum loading time (prevents flash)
        min_loading_ms: u32 = 0,

        /// Key for suspense boundary (for transitions)
        key: ?[]const u8 = null,
    };
}
```

### HTML DSL Integration

```zig
// src/html/suspense.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");
const Element = vdom.Element;

/// Create a suspense boundary
pub fn suspense(
    comptime Msg: type,
    ctx: *AppContext,
    config: SuspenseConfig(Msg),
    children: []const Element(Msg),
) Element(Msg) {
    return .{
        .suspense = .{
            .fallback = config.fallback,
            .children = children,
            .error_fallback = config.error_fallback,
            .min_loading_ms = config.min_loading_ms,
            .key = config.key,
        },
    };
}

pub fn SuspenseConfig(comptime Msg: type) type {
    return struct {
        /// Required: fallback content while loading
        fallback: Element(Msg),

        /// Optional: custom error rendering
        error_fallback: ?*const fn (ResourceError) Element(Msg) = null,

        /// Minimum time to show loading (prevents flash)
        min_loading_ms: u32 = 0,

        /// Key for tracking this suspense boundary
        key: ?[]const u8 = null,
    };
}

/// Convenience: suspense with simple loading spinner
pub fn suspenseWithSpinner(
    comptime Msg: type,
    ctx: *AppContext,
    children: []const Element(Msg),
) Element(Msg) {
    return suspense(Msg, ctx, .{
        .fallback = LoadingSpinner(Msg, ctx),
    }, children);
}

/// Convenience: suspense with skeleton
pub fn suspenseWithSkeleton(
    comptime Msg: type,
    ctx: *AppContext,
    skeleton: Element(Msg),
    children: []const Element(Msg),
) Element(Msg) {
    return suspense(Msg, ctx, .{
        .fallback = skeleton,
    }, children);
}
```

---

## VDOM EXTENSION

### Extended Element Type

```zig
// Addition to src/vdom/element.zig

pub fn Element(comptime Msg: type) type {
    return union(enum) {
        // ... existing variants ...

        /// Suspense boundary for async content
        suspense: struct {
            fallback: *Element(Msg),
            children: []Element(Msg),
            error_fallback: ?*const fn (ResourceError) Element(Msg),
            min_loading_ms: u32,
            key: ?[]const u8,
        },

        /// Await element - suspends until resource is ready
        await_resource: struct {
            resource_key: []const u8,
            render: *const fn (data: anytype) Element(Msg),
            /// If true, shows fallback. If false, uses last data.
            suspend_on_revalidate: bool,
        },
    };
}
```

### Await Helper

```zig
// src/html/await.zig

/// Await a resource - suspends parent Suspense boundary if loading
pub fn await_(
    comptime T: type,
    comptime Msg: type,
    ctx: *AppContext,
    resource: Resource(T),
    render: *const fn (T, *AppContext) Element(Msg),
) Element(Msg) {
    return switch (resource) {
        .idle, .loading => .{ .suspend_marker = {} },
        .success => |data| render(data, ctx),
        .revalidating => |data| render(data, ctx),
        .failure => |err| .{
            .error_marker = .{ .error = err },
        },
    };
}

/// Await with custom loading/error handling inline
pub fn awaitWithStates(
    comptime T: type,
    comptime Msg: type,
    ctx: *AppContext,
    resource: Resource(T),
    handlers: AwaitHandlers(T, Msg),
) Element(Msg) {
    return switch (resource) {
        .idle => handlers.on_idle(ctx),
        .loading => handlers.on_loading(ctx),
        .success => |data| handlers.on_success(data, ctx),
        .revalidating => |data| handlers.on_revalidating(data, ctx),
        .failure => |err| handlers.on_error(err, ctx),
    };
}

pub fn AwaitHandlers(comptime T: type, comptime Msg: type) type {
    return struct {
        on_idle: *const fn (*AppContext) Element(Msg) = defaultIdle,
        on_loading: *const fn (*AppContext) Element(Msg) = defaultLoading,
        on_success: *const fn (T, *AppContext) Element(Msg),
        on_revalidating: *const fn (T, *AppContext) Element(Msg) = null,
        on_error: *const fn (ResourceError, *AppContext) Element(Msg) = defaultError,

        fn defaultIdle(ctx: *AppContext) Element(Msg) {
            return .none;
        }

        fn defaultLoading(ctx: *AppContext) Element(Msg) {
            return .{ .suspend_marker = {} };
        }

        fn defaultError(err: ResourceError, ctx: *AppContext) Element(Msg) {
            return .{ .error_marker = .{ .error = err } };
        }
    };
}
```

---

## SUSPENSE RESOLUTION

### Runtime Behavior

```zig
// src/suspense/resolver.zig

pub const SuspenseResolver = struct {
    pending_boundaries: std.StringHashMap(BoundaryState),
    min_loading_timers: std.StringHashMap(u64),

    pub const BoundaryState = struct {
        is_suspended: bool,
        start_time: u64,
        min_loading_ms: u32,
        children_states: []ChildState,
    };

    pub const ChildState = union(enum) {
        ready,
        suspended: []const u8, // resource key
        error: ResourceError,
    };

    /// Check if a suspense boundary should show fallback
    pub fn shouldShowFallback(self: *SuspenseResolver, boundary_key: []const u8) bool {
        const state = self.pending_boundaries.get(boundary_key) orelse return false;

        if (!state.is_suspended) return false;

        // Check minimum loading time
        if (state.min_loading_ms > 0) {
            const elapsed = getCurrentTime() - state.start_time;
            if (elapsed < state.min_loading_ms) {
                // Still within minimum loading time, show fallback
                return true;
            }
        }

        // Check if any children are still suspended
        for (state.children_states) |child| {
            if (child == .suspended) return true;
        }

        return false;
    }

    /// Mark a resource as loaded
    pub fn markReady(self: *SuspenseResolver, resource_key: []const u8) void {
        var iter = self.pending_boundaries.iterator();
        while (iter.next()) |entry| {
            for (entry.value_ptr.children_states) |*child| {
                if (child.* == .suspended and
                    std.mem.eql(u8, child.suspended, resource_key))
                {
                    child.* = .ready;
                }
            }

            // Check if boundary can be resolved
            self.tryResolveBoundary(entry.key_ptr.*);
        }
    }

    fn tryResolveBoundary(self: *SuspenseResolver, boundary_key: []const u8) void {
        const state = self.pending_boundaries.get(boundary_key) orelse return;

        // Check if all children are ready
        for (state.children_states) |child| {
            if (child != .ready) return;
        }

        // Check minimum loading time
        if (state.min_loading_ms > 0) {
            const elapsed = getCurrentTime() - state.start_time;
            if (elapsed < state.min_loading_ms) {
                // Schedule resolution after remaining time
                const remaining = state.min_loading_ms - elapsed;
                scheduleResolution(boundary_key, remaining);
                return;
            }
        }

        // Boundary resolved - trigger re-render
        state.is_suspended = false;
    }
};
```

---

## USAGE EXAMPLES

### Basic Suspense

```zig
const zui = @import("zui");
const h = zui.html;
const Resource = zui.suspense.Resource;

pub fn UserProfile(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return h.suspense(Msg, ctx, .{
        .fallback = h.div(ctx, .{
            .class = "loading-skeleton",
        }, &.{
            h.div(ctx, .{ .class = "skeleton-avatar" }, &.{}),
            h.div(ctx, .{ .class = "skeleton-text" }, &.{}),
        }),
    }, &.{
        // This will suspend if user is loading
        h.await_(User, Msg, ctx, model.user, renderUser),
    });
}

fn renderUser(user: User, ctx: *zui.AppContext) zui.Element(Msg) {
    return h.div(ctx, .{ .class = "user-profile" }, &.{
        h.img(ctx, .{ .src = user.avatar, .alt = user.name }),
        h.h2(ctx, .{}, &.{ h.text(ctx, user.name) }),
        h.p(ctx, .{}, &.{ h.text(ctx, user.bio) }),
    });
}
```

### Nested Suspense

```zig
pub fn Dashboard(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return h.div(ctx, .{ .class = "dashboard" }, &.{
        // Outer suspense for layout
        h.suspense(Msg, ctx, .{
            .fallback = DashboardSkeleton(ctx),
        }, &.{
            h.await_(DashboardData, Msg, ctx, model.dashboard, renderDashboard),
        }),
    });
}

fn renderDashboard(data: DashboardData, ctx: *zui.AppContext) zui.Element(Msg) {
    return h.div(ctx, .{ .class = "dashboard-content" }, &.{
        // Header loads first
        h.header(ctx, .{}, &.{
            h.text(ctx, data.title),
        }),

        // Nested suspense for widgets - can load independently
        h.suspense(Msg, ctx, .{
            .fallback = WidgetSkeleton(ctx),
        }, &.{
            h.await_([]Widget, Msg, ctx, model.widgets, renderWidgets),
        }),

        // Another independent section
        h.suspense(Msg, ctx, .{
            .fallback = ChartSkeleton(ctx),
        }, &.{
            h.await_(ChartData, Msg, ctx, model.chart, renderChart),
        }),
    });
}
```

### Suspense with Error Handling

```zig
pub fn UserList(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return h.suspense(Msg, ctx, .{
        .fallback = LoadingSpinner(ctx),
        .error_fallback = renderError,
    }, &.{
        h.await_([]User, Msg, ctx, model.users, renderUsers),
    });
}

fn renderError(err: zui.suspense.ResourceError, ctx: *zui.AppContext) zui.Element(Msg) {
    return h.div(ctx, .{ .class = "error-container" }, &.{
        h.p(ctx, .{ .class = "error-message" }, &.{
            h.text(ctx, switch (err.code) {
                .network => "Network error. Please check your connection.",
                .timeout => "Request timed out. Please try again.",
                .not_found => "Data not found.",
                .server_error => "Server error. Please try again later.",
                else => err.message,
            }),
        }),
        h.button(ctx, .{
            .onClick = .retry_fetch,
            .class = "retry-button",
        }, &.{
            h.text(ctx, "Retry"),
        }),
    });
}
```

### Minimum Loading Time (Prevent Flash)

```zig
pub fn QuickData(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    // Prevent loading flash for fast responses
    return h.suspense(Msg, ctx, .{
        .fallback = LoadingSpinner(ctx),
        .min_loading_ms = 200, // Show loading for at least 200ms
    }, &.{
        h.await_(Data, Msg, ctx, model.data, renderData),
    });
}
```

### Inline State Handling

```zig
pub fn InlineExample(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    // Handle all states inline without Suspense boundary
    return h.awaitWithStates(User, Msg, ctx, model.user, .{
        .on_idle = struct {
            fn f(c: *zui.AppContext) zui.Element(Msg) {
                return h.button(c, .{ .onClick = .load_user }, &.{
                    h.text(c, "Load User"),
                });
            }
        }.f,
        .on_loading = struct {
            fn f(c: *zui.AppContext) zui.Element(Msg) {
                return LoadingSpinner(c);
            }
        }.f,
        .on_success = renderUser,
        .on_error = struct {
            fn f(err: ResourceError, c: *zui.AppContext) zui.Element(Msg) {
                return h.text(c, "Failed to load user");
            }
        }.f,
    });
}
```

---

## INTEGRATION WITH EFFECTS

### Fetching Data

```zig
pub fn update(model: *Model, msg: Msg, ctx: *zui.AppContext) zui.Effect(Msg) {
    return switch (msg) {
        .fetch_users => blk: {
            model.users = .loading;
            break :blk zui.Effect(Msg).http(.{
                .method = .GET,
                .url = "/api/users",
                .on_success = parseUsers,
                .on_error = handleUsersError,
            });
        },
        .users_loaded => |users| blk: {
            model.users = .{ .success = users };
            break :blk .none;
        },
        .users_error => |err| blk: {
            model.users = .{ .failure = err };
            break :blk .none;
        },
        .retry_fetch => update(model, .fetch_users, ctx),
        else => .none,
    };
}

fn parseUsers(response: []const u8) Msg {
    const users = std.json.parse([]User, response, .{}) catch {
        return .{ .users_error = .{ .code = .parse_error, .message = "Invalid JSON" } };
    };
    return .{ .users_loaded = users };
}

fn handleUsersError(err: zui.HttpError) Msg {
    return .{
        .users_error = .{
            .code = switch (err) {
                .network_error => .network,
                .timeout => .timeout,
                .status_error => |code| if (code == 404) .not_found else .server_error,
                else => .unknown,
            },
            .message = "Failed to fetch users",
        },
    };
}
```

---

## COMMON PATTERNS

### Loading Skeletons

```zig
// src/components/skeletons.zig

pub fn TextSkeleton(ctx: *zui.AppContext, lines: u8) zui.Element(Msg) {
    var children: [10]zui.Element(Msg) = undefined;
    const count = @min(lines, 10);

    for (0..count) |i| {
        children[i] = h.div(ctx, .{
            .class = "skeleton-line",
            .style = zui.css.styles.create(.{
                .height = .{ .rem = 1 },
                .background = .{ .hex = 0xe0e0e0 },
                .border_radius = .{ .px = 4 },
                .margin_bottom = .{ .rem = 0.5 },
                .width = if (i == count - 1) .{ .percent = 60 } else .{ .percent = 100 },
            }),
        }, &.{});
    }

    return h.div(ctx, .{ .class = "text-skeleton" }, children[0..count]);
}

pub fn CardSkeleton(ctx: *zui.AppContext) zui.Element(Msg) {
    return h.div(ctx, .{
        .class = "card-skeleton",
        .style = zui.css.styles.create(.{
            .padding = .{ .rem = 1 },
            .border = .{ .width = .{ .px = 1 }, .color = .{ .hex = 0xe0e0e0 } },
            .border_radius = .{ .px = 8 },
        }),
    }, &.{
        h.div(ctx, .{
            .class = "skeleton-avatar",
            .style = zui.css.styles.create(.{
                .width = .{ .px = 48 },
                .height = .{ .px = 48 },
                .border_radius = .{ .percent = 50 },
                .background = .{ .hex = 0xe0e0e0 },
            }),
        }, &.{}),
        TextSkeleton(ctx, 3),
    });
}

pub fn TableSkeleton(ctx: *zui.AppContext, rows: u8) zui.Element(Msg) {
    // ... similar pattern
}
```

### Prefetching

```zig
// Prefetch data before navigation
pub fn prefetchUserProfile(user_id: u32) zui.Effect(Msg) {
    return zui.Effect(Msg).prefetch(.{
        .key = std.fmt.allocPrint("user:{d}", .{user_id}),
        .fetcher = struct {
            fn f() zui.Effect(Msg) {
                return zui.Effect(Msg).http(.{
                    .url = std.fmt.allocPrint("/api/users/{d}", .{user_id}),
                    .on_success = parseUser,
                    .on_error = handleError,
                });
            }
        }.f,
    });
}
```

---

## JS RUNTIME INTEGRATION

```javascript
// www/zui.js - Suspense handling

const ZuiSuspense = {
    boundaries: new Map(),
    minLoadingTimers: new Map(),

    registerBoundary(key, minLoadingMs) {
        this.boundaries.set(key, {
            suspended: true,
            startTime: performance.now(),
            minLoadingMs,
        });
    },

    markReady(resourceKey) {
        // Notify WASM that resource is ready
        ZuiRuntime.wasm.exports.suspense_resource_ready(
            ...ZuiRuntime.strToPtr(resourceKey)
        );
    },

    shouldShowFallback(boundaryKey) {
        const boundary = this.boundaries.get(boundaryKey);
        if (!boundary || !boundary.suspended) return false;

        const elapsed = performance.now() - boundary.startTime;
        return elapsed < boundary.minLoadingMs;
    },

    scheduleResolution(boundaryKey, delayMs) {
        if (this.minLoadingTimers.has(boundaryKey)) {
            clearTimeout(this.minLoadingTimers.get(boundaryKey));
        }

        const timer = setTimeout(() => {
            ZuiRuntime.wasm.exports.suspense_boundary_resolve(
                ...ZuiRuntime.strToPtr(boundaryKey)
            );
            this.minLoadingTimers.delete(boundaryKey);
        }, delayMs);

        this.minLoadingTimers.set(boundaryKey, timer);
    },
};
```

---

## CONCLUSION

The Suspense system provides a clean, declarative way to handle loading states in ZUI applications. It integrates seamlessly with the existing Effects system and VDOM, allowing developers to create smooth loading experiences with minimal boilerplate.

**Links:**
- [← Previous: Tailwind Integration](21-tailwind-integration.md)
- [→ Next: Portals](23-portals.md)
- [↑ Back to index](../README.md)
