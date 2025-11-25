# 11 - EFFECTS SYSTEM

**Document:** Effects System for Asynchronous Operations
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

The ZUI effects system handles side effects and asynchronous operations in a type-safe manner, separating pure logic from I/O.

---

## ARCHITECTURE

### Effect Type

```zig
// src/effects/effect.zig
const std = @import("std");

pub fn Effect(comptime Msg: type) type {
    return union(enum) {
        /// No effect
        none,

        /// Batch multiple effects
        batch: []Effect(Msg),

        /// HTTP request
        http: struct {
            method: HttpMethod,
            url: []const u8,
            headers: ?std.StringHashMap([]const u8),
            body: ?[]const u8,
            on_success: fn ([]const u8) Msg,
            on_error: fn (HttpError) Msg,
        },

        /// Timer/Timeout
        timeout: struct {
            duration_ms: u32,
            on_complete: Msg,
        },

        /// Interval
        interval: struct {
            duration_ms: u32,
            on_tick: Msg,
        },

        /// Local storage
        local_storage: union(enum) {
            get: struct {
                key: []const u8,
                on_success: fn (?[]const u8) Msg,
            },
            set: struct {
                key: []const u8,
                value: []const u8,
                on_complete: Msg,
            },
            remove: struct {
                key: []const u8,
                on_complete: Msg,
            },
        },

        /// Random number generation
        random: struct {
            min: i32,
            max: i32,
            on_value: fn (i32) Msg,
        },

        /// Current time
        now: struct {
            on_time: fn (u64) Msg,
        },

        /// Navigation
        navigate: struct {
            path: []const u8,
            on_complete: ?Msg,
        },

        /// Focus element
        focus: struct {
            element_id: []const u8,
        },

        /// Custom effect with JS interop
        custom: struct {
            name: []const u8,
            data: []const u8,
            on_success: fn ([]const u8) Msg,
            on_error: fn ([]const u8) Msg,
        },
    };
}

pub const HttpMethod = enum {
    GET,
    POST,
    PUT,
    PATCH,
    DELETE,
};

pub const HttpError = union(enum) {
    network_error: []const u8,
    timeout,
    status_error: u16,
    parse_error: []const u8,
};
```

### Effect Constructors

```zig
/// No effect
pub fn none(comptime Msg: type) Effect(Msg) {
    return .none;
}

/// Batch multiple effects
pub fn batch(comptime Msg: type, effects: []Effect(Msg)) Effect(Msg) {
    return .{ .batch = effects };
}

/// HTTP GET request
pub fn get(
    comptime Msg: type,
    url: []const u8,
    on_success: fn ([]const u8) Msg,
    on_error: fn (HttpError) Msg,
) Effect(Msg) {
    return .{
        .http = .{
            .method = .GET,
            .url = url,
            .headers = null,
            .body = null,
            .on_success = on_success,
            .on_error = on_error,
        },
    };
}

/// HTTP POST request
pub fn post(
    comptime Msg: type,
    url: []const u8,
    body: []const u8,
    on_success: fn ([]const u8) Msg,
    on_error: fn (HttpError) Msg,
) Effect(Msg) {
    return .{
        .http = .{
            .method = .POST,
            .url = url,
            .headers = null,
            .body = body,
            .on_success = on_success,
            .on_error = on_error,
        },
    };
}

/// Set timeout
pub fn timeout(comptime Msg: type, duration_ms: u32, on_complete: Msg) Effect(Msg) {
    return .{
        .timeout = .{
            .duration_ms = duration_ms,
            .on_complete = on_complete,
        },
    };
}

/// Set interval
pub fn interval(comptime Msg: type, duration_ms: u32, on_tick: Msg) Effect(Msg) {
    return .{
        .interval = .{
            .duration_ms = duration_ms,
            .on_tick = on_tick,
        },
    };
}
```

---

## EFFECT RUNNER

```zig
// src/effects/runner.zig
const std = @import("std");
const Effect = @import("effect.zig").Effect;

pub const EffectRunner = struct {
    allocator: std.mem.Allocator,
    pending: std.ArrayList(PendingEffect),
    timers: std.AutoHashMap(u32, TimerInfo),
    next_timer_id: u32,

    const PendingEffect = struct {
        effect: anytype,
        msg_type: type,
    };

    const TimerInfo = struct {
        interval: bool,
        msg: anytype,
    };

    pub fn init(allocator: std.mem.Allocator) !EffectRunner {
        return .{
            .allocator = allocator,
            .pending = std.ArrayList(PendingEffect).init(allocator),
            .timers = std.AutoHashMap(u32, TimerInfo).init(allocator),
            .next_timer_id = 0,
        };
    }

    pub fn deinit(self: *EffectRunner) void {
        self.pending.deinit();
        self.timers.deinit();
    }

    /// Schedule an effect for execution
    pub fn schedule(self: *EffectRunner, effect: anytype) !void {
        const Msg = @TypeOf(effect).Msg;

        switch (effect) {
            .none => {},

            .batch => |effects| {
                for (effects) |eff| {
                    try self.schedule(eff);
                }
            },

            .http => |http| {
                try self.scheduleHttp(Msg, http);
            },

            .timeout => |timer| {
                const timer_id = self.next_timer_id;
                self.next_timer_id += 1;

                try self.timers.put(timer_id, .{
                    .interval = false,
                    .msg = timer.on_complete,
                });

                js_setTimeout(timer_id, timer.duration_ms);
            },

            .interval => |timer| {
                const timer_id = self.next_timer_id;
                self.next_timer_id += 1;

                try self.timers.put(timer_id, .{
                    .interval = true,
                    .msg = timer.on_tick,
                });

                js_setInterval(timer_id, timer.duration_ms);
            },

            .local_storage => |storage| {
                try self.scheduleLocalStorage(Msg, storage);
            },

            .navigate => |nav| {
                js_navigate(nav.path.ptr, nav.path.len);
            },

            .focus => |focus| {
                js_focusElement(focus.element_id.ptr, focus.element_id.len);
            },

            .custom => |custom| {
                try self.scheduleCustom(Msg, custom);
            },

            else => {},
        }
    }

    fn scheduleHttp(self: *EffectRunner, comptime Msg: type, http: anytype) !void {
        const method_str = @tagName(http.method);

        js_httpRequest(
            method_str.ptr,
            method_str.len,
            http.url.ptr,
            http.url.len,
            if (http.body) |b| b.ptr else null,
            if (http.body) |b| b.len else 0,
        );
    }

    /// Handle timer completion
    pub export fn onTimerComplete(timer_id: u32) void {
        // Dispatch message to app
    }
};

// WASM imports for effects
extern "zui" fn js_setTimeout(timer_id: u32, duration_ms: u32) void;
extern "zui" fn js_setInterval(timer_id: u32, duration_ms: u32) void;
extern "zui" fn js_clearTimer(timer_id: u32) void;
extern "zui" fn js_httpRequest(
    method_ptr: [*]const u8,
    method_len: usize,
    url_ptr: [*]const u8,
    url_len: usize,
    body_ptr: ?[*]const u8,
    body_len: usize,
) void;
extern "zui" fn js_navigate(path_ptr: [*]const u8, path_len: usize) void;
extern "zui" fn js_focusElement(id_ptr: [*]const u8, id_len: usize) void;
```

---

## USAGE IN UPDATE

```zig
pub const Msg = enum {
    fetch_users,
    users_loaded,
    users_failed,
    start_clock,
    tick,
    save_preferences,
    preferences_saved,
};

pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .fetch_users => {
            model.loading = true;
            return Effect.get(
                Msg,
                "/api/users",
                .users_loaded,
                .users_failed,
            );
        },

        .users_loaded => |data| {
            model.loading = false;
            model.users = parseUsers(data);
            return Effect.none(Msg);
        },

        .users_failed => {
            model.loading = false;
            model.error = "Failed to load users";
            return Effect.none(Msg);
        },

        .start_clock => {
            return Effect.interval(Msg, 1000, .tick);
        },

        .tick => {
            model.time += 1;
            return Effect.none(Msg);
        },

        .save_preferences => {
            const json = serializePreferences(model.preferences);
            return Effect.batch(Msg, &.{
                Effect.localStorage.set("prefs", json, .preferences_saved),
                Effect.timeout(Msg, 2000, .show_save_confirmation),
            });
        },
    };
}
```

---

## JAVASCRIPT INTEGRATION

```javascript
// www/zui.js - Effects support
const ZuiEffects = {
    // HTTP requests
    httpRequest(methodPtr, methodLen, urlPtr, urlLen, bodyPtr, bodyLen) {
        const method = ptrToStr(methodPtr, methodLen);
        const url = ptrToStr(urlPtr, urlLen);
        const body = bodyPtr ? ptrToStr(bodyPtr, bodyLen) : null;

        const options = {
            method,
            headers: { 'Content-Type': 'application/json' }
        };

        if (body) options.body = body;

        fetch(url, options)
            .then(r => r.text())
            .then(data => {
                wasmExports.effect_httpSuccess(
                    strToPtr(data),
                    data.length
                );
            })
            .catch(err => {
                const msg = err.message;
                wasmExports.effect_httpError(
                    strToPtr(msg),
                    msg.length
                );
            });
    },

    // Timers
    setTimeout(timerId, durationMs) {
        setTimeout(() => {
            wasmExports.onTimerComplete(timerId);
        }, durationMs);
    },

    setInterval(timerId, durationMs) {
        setInterval(() => {
            wasmExports.onTimerComplete(timerId);
        }, durationMs);
    },

    // Local storage
    localStorageGet(keyPtr, keyLen) {
        const key = ptrToStr(keyPtr, keyLen);
        const value = localStorage.getItem(key);

        if (value) {
            wasmExports.effect_storageValue(
                strToPtr(value),
                value.length
            );
        } else {
            wasmExports.effect_storageValue(0, 0);
        }
    },

    localStorageSet(keyPtr, keyLen, valuePtr, valueLen) {
        const key = ptrToStr(keyPtr, keyLen);
        const value = ptrToStr(valuePtr, valueLen);
        localStorage.setItem(key, value);
    }
};
```

---

## DATA FETCHING & CACHING

### Query System

Advanced data fetching with automatic caching, deduplication, and background refresh:

```zig
// src/effects/query.zig
const std = @import("std");
const Effect = @import("effect.zig").Effect;

/// Query state
pub const QueryState = enum {
    idle,
    loading,
    success,
    error,
};

/// Query key for cache identification
pub const QueryKey = struct {
    parts: []const []const u8,

    pub fn hash(self: QueryKey) u64 {
        var hasher = std.hash.Wyhash.init(0);
        for (self.parts) |part| {
            hasher.update(part);
        }
        return hasher.final();
    }

    pub fn equal(self: QueryKey, other: QueryKey) bool {
        if (self.parts.len != other.parts.len) return false;
        for (self.parts, other.parts) |a, b| {
            if (!std.mem.eql(u8, a, b)) return false;
        }
        return true;
    }
};

/// Cached query data
pub const CachedQuery = struct {
    data: []const u8,
    timestamp: u64,
    stale_time: u32,  // ms
    cache_time: u32,  // ms

    pub fn isStale(self: CachedQuery, now: u64) bool {
        return (now - self.timestamp) > self.stale_time;
    }

    pub fn isExpired(self: CachedQuery, now: u64) bool {
        return (now - self.timestamp) > self.cache_time;
    }
};

/// Query cache manager
pub const QueryCache = struct {
    allocator: std.mem.Allocator,
    cache: std.AutoHashMap(u64, CachedQuery),
    in_flight: std.AutoHashMap(u64, void),  // Track ongoing requests

    pub fn init(allocator: std.mem.Allocator) QueryCache {
        return .{
            .allocator = allocator,
            .cache = std.AutoHashMap(u64, CachedQuery).init(allocator),
            .in_flight = std.AutoHashMap(u64, void).init(allocator),
        };
    }

    pub fn deinit(self: *QueryCache) void {
        self.cache.deinit();
        self.in_flight.deinit();
    }

    /// Get cached data if available and not expired
    pub fn get(self: *QueryCache, key: QueryKey, now: u64) ?CachedQuery {
        const hash_key = key.hash();
        const cached = self.cache.get(hash_key) orelse return null;

        if (cached.isExpired(now)) {
            _ = self.cache.remove(hash_key);
            return null;
        }

        return cached;
    }

    /// Set cached data
    pub fn set(self: *QueryCache, key: QueryKey, data: []const u8, opts: struct {
        stale_time: u32 = 5000,  // 5 seconds default
        cache_time: u32 = 300000,  // 5 minutes default
    }, now: u64) !void {
        const hash_key = key.hash();
        try self.cache.put(hash_key, .{
            .data = data,
            .timestamp = now,
            .stale_time = opts.stale_time,
            .cache_time = opts.cache_time,
        });
    }

    /// Mark query as in-flight (ongoing request)
    pub fn markInFlight(self: *QueryCache, key: QueryKey) !void {
        const hash_key = key.hash();
        try self.in_flight.put(hash_key, {});
    }

    /// Check if query is in-flight
    pub fn isInFlight(self: *QueryCache, key: QueryKey) bool {
        const hash_key = key.hash();
        return self.in_flight.contains(hash_key);
    }

    /// Clear in-flight marker
    pub fn clearInFlight(self: *QueryCache, key: QueryKey) void {
        const hash_key = key.hash();
        _ = self.in_flight.remove(hash_key);
    }

    /// Invalidate (remove) cached query
    pub fn invalidate(self: *QueryCache, key: QueryKey) void {
        const hash_key = key.hash();
        _ = self.cache.remove(hash_key);
    }

    /// Clear all cached queries
    pub fn clear(self: *QueryCache) void {
        self.cache.clearRetainingCapacity();
        self.in_flight.clearRetainingCapacity();
    }
};
```

### Query Effect

```zig
// Extended Effect type with query support
pub fn Effect(comptime Msg: type) type {
    return union(enum) {
        // ... existing effects ...

        /// Query with caching and deduplication
        query: struct {
            key: QueryKey,
            url: []const u8,
            method: HttpMethod = .GET,
            body: ?[]const u8 = null,
            stale_time: u32 = 5000,   // ms before data considered stale
            cache_time: u32 = 300000, // ms before data removed from cache
            refetch_on_stale: bool = true,
            on_success: fn ([]const u8) Msg,
            on_error: fn (HttpError) Msg,
        },

        /// Invalidate query cache
        invalidate_query: struct {
            key: QueryKey,
        },

        /// Prefetch query
        prefetch_query: struct {
            key: QueryKey,
            url: []const u8,
        },
    };
}
```

### Query Helpers

```zig
// src/effects/query_helpers.zig
pub const Query = struct {
    /// Create a query effect with caching
    pub fn fetch(
        comptime Msg: type,
        key: []const []const u8,
        url: []const u8,
        opts: struct {
            stale_time: u32 = 5000,
            cache_time: u32 = 300000,
            refetch_on_stale: bool = true,
        },
        on_success: fn ([]const u8) Msg,
        on_error: fn (HttpError) Msg,
    ) Effect(Msg) {
        return .{
            .query = .{
                .key = .{ .parts = key },
                .url = url,
                .method = .GET,
                .stale_time = opts.stale_time,
                .cache_time = opts.cache_time,
                .refetch_on_stale = opts.refetch_on_stale,
                .on_success = on_success,
                .on_error = on_error,
            },
        };
    }

    /// Invalidate a query
    pub fn invalidate(comptime Msg: type, key: []const []const u8) Effect(Msg) {
        return .{
            .invalidate_query = .{
                .key = .{ .parts = key },
            },
        };
    }

    /// Prefetch a query
    pub fn prefetch(comptime Msg: type, key: []const []const u8, url: []const u8) Effect(Msg) {
        return .{
            .prefetch_query = .{
                .key = .{ .parts = key },
                .url = url,
            },
        };
    }
};
```

### Query Usage Example

```zig
// Example: User data fetching with caching
pub const Model = struct {
    user_id: u32,
    user_data: ?UserData,
    user_state: QueryState,
    user_error: ?[]const u8,
};

pub const Msg = union(enum) {
    fetch_user: u32,
    user_loaded: []const u8,
    user_failed: HttpError,
    refresh_user,
    invalidate_user,
};

pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .fetch_user => |user_id| {
            model.user_state = .loading;
            model.user_id = user_id;

            // Query with automatic caching and deduplication
            const url = std.fmt.allocPrint(ctx.allocator, "/api/users/{d}", .{user_id}) catch "";

            return Query.fetch(
                Msg,
                &.{ "user", std.fmt.allocPrint(ctx.allocator, "{d}", .{user_id}) catch "" },
                url,
                .{
                    .stale_time = 30000,      // 30 seconds
                    .cache_time = 300000,     // 5 minutes
                    .refetch_on_stale = true, // Refetch in background when stale
                },
                |data| .{ .user_loaded = data },
                |err| .{ .user_failed = err },
            );
        },

        .user_loaded => |data| {
            model.user_state = .success;
            model.user_data = parseUserData(data);
            return Effect.none(Msg);
        },

        .user_failed => |error| {
            model.user_state = .error;
            model.user_error = formatError(error);
            return Effect.none(Msg);
        },

        .refresh_user => {
            // Invalidate cache and refetch
            return Effect.batch(Msg, &.{
                Query.invalidate(Msg, &.{ "user", std.fmt.allocPrint(ctx.allocator, "{d}", .{model.user_id}) catch "" }),
                .{ .fetch_user = model.user_id },
            });
        },

        .invalidate_user => {
            return Query.invalidate(Msg, &.{ "user", std.fmt.allocPrint(ctx.allocator, "{d}", .{model.user_id}) catch "" });
        },
    };
}
```

### Mutation with Optimistic Updates

```zig
// Mutation effect for updates
pub const Mutation = struct {
    /// Mutate data with optimistic updates
    pub fn mutate(
        comptime Msg: type,
        url: []const u8,
        method: HttpMethod,
        body: []const u8,
        opts: struct {
            optimistic_update: ?fn (*Model) void = null,
            invalidate_keys: []const QueryKey = &.{},
        },
        on_success: fn ([]const u8) Msg,
        on_error: fn (HttpError) Msg,
    ) Effect(Msg) {
        return .{
            .mutation = .{
                .url = url,
                .method = method,
                .body = body,
                .optimistic_update = opts.optimistic_update,
                .invalidate_keys = opts.invalidate_keys,
                .on_success = on_success,
                .on_error = on_error,
            },
        };
    }
};

// Extended Effect type
pub fn Effect(comptime Msg: type) type {
    return union(enum) {
        // ... existing effects ...

        /// Mutation with optimistic updates
        mutation: struct {
            url: []const u8,
            method: HttpMethod,
            body: []const u8,
            optimistic_update: ?fn (*Model) void,
            invalidate_keys: []const QueryKey,
            on_success: fn ([]const u8) Msg,
            on_error: fn (HttpError) Msg,
            on_rollback: ?Msg,  // Message to dispatch on error
        },
    };
}
```

### Mutation Example

```zig
// Example: Update user with optimistic UI
pub const Msg = union(enum) {
    update_user_name: []const u8,
    update_user_success: []const u8,
    update_user_failed: HttpError,
};

pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .update_user_name => |new_name| {
            // Save current name for rollback
            const old_name = model.user_data.?.name;

            // Optimistically update UI
            model.user_data.?.name = new_name;

            const body = std.fmt.allocPrint(
                ctx.allocator,
                "{{\"name\":\"{s}\"}}",
                .{new_name},
            ) catch "";

            return Mutation.mutate(
                Msg,
                "/api/users/me",
                .PUT,
                body,
                .{
                    .invalidate_keys = &.{
                        .{ .parts = &.{ "user", "me" } },
                    },
                },
                |data| .{ .update_user_success = data },
                |err| .{ .update_user_failed = err },
            );
        },

        .update_user_success => {
            // Mutation confirmed, nothing to do
            return Effect.none(Msg);
        },

        .update_user_failed => |error| {
            // Rollback optimistic update
            // (In real app, would store old value properly)
            model.user_error = "Failed to update name";
            return Effect.none(Msg);
        },
    };
}
```

### Pagination Support

```zig
// Pagination helper
pub const Pagination = struct {
    /// Paginated query
    pub fn query(
        comptime Msg: type,
        base_key: []const []const u8,
        url_template: []const u8,
        page: u32,
        opts: struct {
            page_size: u32 = 20,
            stale_time: u32 = 30000,
            cache_time: u32 = 300000,
        },
        on_success: fn ([]const u8) Msg,
        on_error: fn (HttpError) Msg,
    ) Effect(Msg) {
        // Create page-specific key
        const page_str = std.fmt.allocPrint(ctx.allocator, "{d}", .{page}) catch "";
        var key_parts = std.ArrayList([]const u8).init(ctx.allocator);
        key_parts.appendSlice(base_key) catch {};
        key_parts.append(page_str) catch {};

        const url = std.fmt.allocPrint(
            ctx.allocator,
            url_template,
            .{ page, opts.page_size },
        ) catch "";

        return Query.fetch(
            Msg,
            key_parts.items,
            url,
            .{
                .stale_time = opts.stale_time,
                .cache_time = opts.cache_time,
            },
            on_success,
            on_error,
        );
    }

    /// Infinite scroll query
    pub fn infinite(
        comptime Msg: type,
        base_key: []const []const u8,
        url_template: []const u8,
        cursor: ?[]const u8,
        on_success: fn ([]const u8) Msg,
        on_error: fn (HttpError) Msg,
    ) Effect(Msg) {
        const cursor_str = cursor orelse "0";
        var key_parts = std.ArrayList([]const u8).init(ctx.allocator);
        key_parts.appendSlice(base_key) catch {};
        key_parts.append("infinite") catch {};
        key_parts.append(cursor_str) catch {};

        const url = std.fmt.allocPrint(
            ctx.allocator,
            url_template,
            .{cursor_str},
        ) catch "";

        return Query.fetch(
            Msg,
            key_parts.items,
            url,
            .{},
            on_success,
            on_error,
        );
    }
};
```

### Pagination Example

```zig
// Example: Paginated list
pub const Model = struct {
    current_page: u32,
    users: []User,
    users_state: QueryState,
    has_next_page: bool,
};

pub const Msg = union(enum) {
    load_page: u32,
    page_loaded: []const u8,
    page_failed: HttpError,
};

pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .load_page => |page| {
            model.current_page = page;
            model.users_state = .loading;

            return Pagination.query(
                Msg,
                &.{"users"},
                "/api/users?page={d}&size={d}",
                page,
                .{
                    .page_size = 20,
                    .stale_time = 60000,  // 1 minute
                },
                |data| .{ .page_loaded = data },
                |err| .{ .page_failed = err },
            );
        },

        .page_loaded => |data| {
            model.users_state = .success;
            const result = parseUsersPage(data);
            model.users = result.users;
            model.has_next_page = result.has_next;
            return Effect.none(Msg);
        },

        .page_failed => |error| {
            model.users_state = .error;
            return Effect.none(Msg);
        },
    };
}
```

### Cache Management in AppContext

```zig
// Add query cache to AppContext
pub const AppContext = struct {
    // ... existing fields ...
    query_cache: QueryCache,

    pub fn init(allocator: std.mem.Allocator) !AppContext {
        return .{
            // ... existing initialization ...
            .query_cache = QueryCache.init(allocator),
        };
    }

    pub fn deinit(self: *AppContext) void {
        // ... existing cleanup ...
        self.query_cache.deinit();
    }
};
```

---

## CONCLUSION

The effects system provides a type-safe and composable way to handle side effects in ZUI applications. The query system adds powerful data fetching capabilities with automatic caching, request deduplication, background refresh, optimistic updates, and pagination support - bringing ZUI on par with modern frameworks like TanStack Query.

**Links:**
- [← Previous: Routing](10-routing.md)
- [→ Next: State Management](12-state-management.md)
- [↑ Back to index](../README.md)
