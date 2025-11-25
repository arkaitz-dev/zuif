# 13 - WASM RUNTIME

**Document:** WebAssembly Runtime
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

The ZUI WASM runtime provides the interface between compiled Zig code and the JavaScript runtime, defining exports for the browser and imports from JavaScript.

---

## WASM EXPORTS

### Core Application Interface

```zig
// src/runtime/wasm.zig
const std = @import("std");
const App = @import("app.zig").App;

/// Global application instance
var app_instance: ?*App = null;
var gpa = std.heap.GeneralPurposeAllocator(.{}){};

/// Initialize the application
export fn init() void {
    const allocator = gpa.allocator();

    app_instance = allocator.create(App) catch |err| {
        logError("Failed to create app", err);
        return;
    };

    app_instance.?.* = App.init(allocator) catch |err| {
        logError("Failed to initialize app", err);
        return;
    };
}

/// Clean up application
export fn deinit() void {
    if (app_instance) |app| {
        app.deinit();
        gpa.allocator().destroy(app);
        app_instance = null;
    }
    _ = gpa.deinit();
}

/// Render the application
export fn render() void {
    if (app_instance) |app| {
        app.render() catch |err| {
            logError("Render failed", err);
        };
    }
}

/// Dispatch a message to the application
export fn dispatch(msg_id: u32) void {
    if (app_instance) |app| {
        app.dispatch(msg_id) catch |err| {
            logError("Dispatch failed", err);
        };
    }
}

/// Dispatch message with string payload
export fn dispatchString(msg_id: u32, data_ptr: [*]const u8, data_len: usize) void {
    if (app_instance) |app| {
        const data = data_ptr[0..data_len];
        app.dispatchString(msg_id, data) catch |err| {
            logError("Dispatch string failed", err);
        };
    }
}

/// Handle route change from browser
export fn router_onPopState(path_ptr: [*]const u8, path_len: usize) void {
    if (app_instance) |app| {
        const path = path_ptr[0..path_len];
        app.onRouteChange(path) catch |err| {
            logError("Route change failed", err);
        };
    }
}

/// Handle HTTP success response
export fn effect_httpSuccess(data_ptr: [*]const u8, data_len: usize) void {
    if (app_instance) |app| {
        const data = data_ptr[0..data_len];
        app.onHttpSuccess(data) catch |err| {
            logError("HTTP success handler failed", err);
        };
    }
}

/// Handle HTTP error response
export fn effect_httpError(error_ptr: [*]const u8, error_len: usize) void {
    if (app_instance) |app| {
        const error_msg = error_ptr[0..error_len];
        app.onHttpError(error_msg) catch |err| {
            logError("HTTP error handler failed", err);
        };
    }
}

/// Handle timer completion
export fn onTimerComplete(timer_id: u32) void {
    if (app_instance) |app| {
        app.onTimerComplete(timer_id) catch |err| {
            logError("Timer handler failed", err);
        };
    }
}

/// Handle localStorage value retrieval
export fn effect_storageValue(value_ptr: [*]const u8, value_len: usize) void {
    if (app_instance) |app| {
        if (value_len == 0) {
            app.onStorageValue(null) catch |err| {
                logError("Storage handler failed", err);
            };
        } else {
            const value = value_ptr[0..value_len];
            app.onStorageValue(value) catch |err| {
                logError("Storage handler failed", err);
            };
        }
    }
}
```

---

## WASM IMPORTS

### JavaScript Interop

```zig
/// DOM Operations
extern "zui" fn js_createElement(tag_ptr: [*]const u8, tag_len: usize) u32;
extern "zui" fn js_createTextNode(text_ptr: [*]const u8, text_len: usize) u32;
extern "zui" fn js_appendChild(parent_id: u32, child_id: u32) void;
extern "zui" fn js_removeChild(parent_id: u32, child_id: u32) void;
extern "zui" fn js_replaceChild(parent_id: u32, old_id: u32, new_id: u32) void;
extern "zui" fn js_insertBefore(parent_id: u32, new_id: u32, ref_id: u32) void;
extern "zui" fn js_setAttribute(
    node_id: u32,
    key_ptr: [*]const u8,
    key_len: usize,
    val_ptr: [*]const u8,
    val_len: usize,
) void;
extern "zui" fn js_removeAttribute(node_id: u32, key_ptr: [*]const u8, key_len: usize) void;
extern "zui" fn js_setTextContent(node_id: u32, text_ptr: [*]const u8, text_len: usize) void;
extern "zui" fn js_addEventListener(
    node_id: u32,
    event_ptr: [*]const u8,
    event_len: usize,
    msg_id: u32,
) void;

/// Effects - HTTP
extern "zui" fn js_httpRequest(
    method_ptr: [*]const u8,
    method_len: usize,
    url_ptr: [*]const u8,
    url_len: usize,
    body_ptr: ?[*]const u8,
    body_len: usize,
) void;

/// Effects - Timers
extern "zui" fn js_setTimeout(timer_id: u32, duration_ms: u32) void;
extern "zui" fn js_setInterval(timer_id: u32, duration_ms: u32) void;
extern "zui" fn js_clearTimer(timer_id: u32) void;

/// Effects - LocalStorage
extern "zui" fn js_localStorageGet(key_ptr: [*]const u8, key_len: usize) void;
extern "zui" fn js_localStorageSet(
    key_ptr: [*]const u8,
    key_len: usize,
    value_ptr: [*]const u8,
    value_len: usize,
) void;
extern "zui" fn js_localStorageRemove(key_ptr: [*]const u8, key_len: usize) void;

/// Effects - Navigation
extern "zui" fn js_pushState(path_ptr: [*]const u8, path_len: usize) void;
extern "zui" fn js_replaceState(path_ptr: [*]const u8, path_len: usize) void;
extern "zui" fn js_goBack() void;
extern "zui" fn js_goForward() void;

/// Effects - Focus
extern "zui" fn js_focusElement(id_ptr: [*]const u8, id_len: usize) void;
extern "zui" fn js_blurElement(id_ptr: [*]const u8, id_len: usize) void;

/// Logging
extern "zui" fn js_log(ptr: [*]const u8, len: usize) void;
extern "zui" fn js_error(ptr: [*]const u8, len: usize) void;
extern "zui" fn js_warn(ptr: [*]const u8, len: usize) void;

/// Random
extern "zui" fn js_random() f64;

/// Time
extern "zui" fn js_now() u64;
```

---

## MEMORY MANAGEMENT

### Linear Memory Layout

```
┌─────────────────────────────────────────┐
│         WASM LINEAR MEMORY              │
├─────────────────────────────────────────┤
│  0x0000 - 0x1000: Stack                 │
│  0x1000 - 0x2000: Data segment          │
│  0x2000 - ...   : Heap (managed by GPA) │
└─────────────────────────────────────────┘
```

### Allocator Exports

```zig
/// Allocate memory (for JS to WASM data transfer)
export fn alloc(size: usize) ?[*]u8 {
    const allocator = gpa.allocator();
    const memory = allocator.alloc(u8, size) catch return null;
    return memory.ptr;
}

/// Free memory
export fn free(ptr: [*]u8, size: usize) void {
    const allocator = gpa.allocator();
    const memory = ptr[0..size];
    allocator.free(memory);
}

/// Get memory size
export fn memory_size() usize {
    return @intFromPtr(std.heap.page_allocator.allocator().alloc(u8, 0) catch return 0);
}
```

### String Transfer Helpers

```zig
/// Copy string from JavaScript to WASM memory
pub fn receiveString(ptr: [*]const u8, len: usize, allocator: std.mem.Allocator) ![]u8 {
    const slice = ptr[0..len];
    return try allocator.dupe(u8, slice);
}

/// Send string from WASM to JavaScript (caller manages memory)
pub fn sendString(s: []const u8) struct { ptr: [*]const u8, len: usize } {
    return .{
        .ptr = s.ptr,
        .len = s.len,
    };
}
```

---

## APPLICATION WRAPPER

### App Structure

```zig
// src/runtime/app.zig
const std = @import("std");
const zui = @import("../main.zig");

pub const App = struct {
    allocator: std.mem.Allocator,
    context: *zui.AppContext,
    model: zui.Model,
    current_vdom: ?zui.Element(zui.Msg),

    pub fn init(allocator: std.mem.Allocator) !App {
        var memory = try zui.MemoryStrategy.init();
        var theme = zui.Theme.default();
        var router = try zui.Router.init(memory.persistent());
        var effects = try zui.EffectRunner.init(memory.persistent());
        var interner = zui.StringInterner.init(memory.persistent());

        var context = try allocator.create(zui.AppContext);
        context.* = zui.AppContext.init(
            &memory.frame_arena,
            &theme,
            &router,
            &effects,
            &interner,
        );

        var model = try zui.Model.init(memory.persistent());

        return .{
            .allocator = allocator,
            .context = context,
            .model = model,
            .current_vdom = null,
        };
    }

    pub fn deinit(self: *App) void {
        self.model.deinit();
        self.context.effects.deinit();
        self.context.router.deinit();
        self.context.interner.deinit();
        self.allocator.destroy(self.context);
    }

    pub fn render(self: *App) !void {
        // Generate new VDOM
        const new_vdom = zui.view(&self.model, self.context);

        // Diff with previous VDOM
        if (self.current_vdom) |old_vdom| {
            const patches = try zui.diff(
                zui.Msg,
                self.context.allocator,
                old_vdom,
                new_vdom,
                0,
            );

            // Apply patches
            try zui.reconcile(zui.Msg, patches);

            self.context.allocator.free(patches);
        } else {
            // Initial render
            try zui.createNode(zui.Msg, 0, new_vdom);
        }

        self.current_vdom = new_vdom;
        self.context.resetFrame();
    }

    pub fn dispatch(self: *App, msg_id: u32) !void {
        const msg = try self.msgFromId(msg_id);
        const effect = zui.update(&self.model, msg, self.context);

        // Schedule effect
        try self.context.effects.schedule(effect);

        // Re-render
        try self.render();
    }

    pub fn dispatchString(self: *App, msg_id: u32, data: []const u8) !void {
        const msg = try self.msgFromIdWithData(msg_id, data);
        const effect = zui.update(&self.model, msg, self.context);

        try self.context.effects.schedule(effect);
        try self.render();
    }

    fn msgFromId(self: *App, id: u32) !zui.Msg {
        // Map message ID to actual message
        // This would be generated by build system
        _ = self;
        return switch (id) {
            0 => .increment,
            1 => .decrement,
            2 => .reset,
            else => error.InvalidMessageId,
        };
    }

    fn msgFromIdWithData(self: *App, id: u32, data: []const u8) !zui.Msg {
        _ = self;
        _ = data;
        // Similar to msgFromId but with payload
        return switch (id) {
            100 => .form_update_name, // would use data
            101 => .form_update_email,
            else => error.InvalidMessageId,
        };
    }

    pub fn onRouteChange(self: *App, path: []const u8) !void {
        try self.context.router.push(path);
        try self.render();
    }

    pub fn onHttpSuccess(self: *App, data: []const u8) !void {
        // Handle HTTP success
        _ = data;
        try self.render();
    }

    pub fn onHttpError(self: *App, error_msg: []const u8) !void {
        // Handle HTTP error
        _ = error_msg;
        try self.render();
    }

    pub fn onTimerComplete(self: *App, timer_id: u32) !void {
        // Handle timer
        _ = timer_id;
        try self.render();
    }

    pub fn onStorageValue(self: *App, value: ?[]const u8) !void {
        // Handle localStorage value
        _ = value;
        try self.render();
    }
};
```

---

## BUILD CONFIGURATION

### Compilation Target

```zig
// build.zig
pub fn build(b: *std.Build) void {
    const target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });

    const optimize = b.standardOptimizeOption(.{});

    const wasm = b.addExecutable(.{
        .name = "app",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    // WASM-specific settings
    wasm.entry = .disabled; // No _start function
    wasm.rdynamic = true;   // Export all symbols
    wasm.import_memory = true;
    wasm.initial_memory = 65536 * 16; // 1MB
    wasm.max_memory = 65536 * 256;    // 16MB
    wasm.stack_size = 65536 * 4;      // 256KB

    b.installArtifact(wasm);
}
```

---

## PERFORMANCE OPTIMIZATIONS

### 1. Memory Pool for VDOM Nodes
```zig
var vdom_pool: std.heap.MemoryPool(VDomNode) = undefined;

pub fn initPools(allocator: std.mem.Allocator) !void {
    vdom_pool = std.heap.MemoryPool(VDomNode).init(allocator);
}
```

### 2. String Interning
```zig
// Reuse common strings (tag names, class names)
const interned_div = try interner.intern("div");
const interned_flex = try interner.intern("flex");
```

### 3. Batch DOM Updates
```zig
// Collect all patches before applying
var patches = std.ArrayList(Patch).init(allocator);
// ... collect patches ...
try applyPatchesBatch(patches.items);
```

---

## DEBUGGING

### Debug Exports

```zig
export fn debug_dumpModel() void {
    if (app_instance) |app| {
        const json = std.json.stringify(app.model, .{}) catch return;
        js_log(json.ptr, json.len);
    }
}

export fn debug_getStats() void {
    if (app_instance) |app| {
        const stats = app.context.effects.getStats();
        // Log stats
    }
}
```

---

## CONCLUSION

The WASM runtime provides the interface between Zig and JavaScript, managing memory, events, and bidirectional communication in an efficient and type-safe manner.

**Links:**
- [← Previous: State Management](12-state-management.md)
- [→ Next: JS Runtime](14-js-runtime.md)
- [↑ Back to index](../README.md)
