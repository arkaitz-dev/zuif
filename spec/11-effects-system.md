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

## CONCLUSION

The effects system provides a type-safe and composable way to handle side effects in ZUI applications.

**Links:**
- [← Previous: Routing](10-routing.md)
- [→ Next: State Management](12-state-management.md)
- [↑ Back to index](../README.md)
