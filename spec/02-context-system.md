# 02 - CONTEXT SYSTEM

**Document:** Context System and Dependency Injection
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

The ZUI context system eliminates all global mutable state, passing dependencies explicitly through an application context. This makes code more testable, predictable, and thread-safe.

---

## MOTIVATION

### Problem: Global State

```zig
// ❌ ANTI-PATTERN: Global mutable state
var global_allocator: std.mem.Allocator = undefined;
var global_theme: Theme = .{};
var global_router: ?*Router = null;

pub fn render() void {
    // Where do these dependencies come from?
    // Are they initialized?
    // What about threading?
    const element = h.div(.{}, &.{});
}
```

### Solution: Context Injection

```zig
// ✅ PATTERN: Explicit context
pub fn render(ctx: *AppContext) Element(Msg) {
    // All dependencies are explicit
    // Thread-safe by design
    // Easy to test
    return h.div(ctx, .{}, &.{});
}
```

---

## APP CONTEXT

### Estructura Principal

```zig
// src/core/context.zig
const std = @import("std");
const Theme = @import("../css/theme.zig").Theme;
const Router = @import("../router/router.zig").Router;
const EffectRunner = @import("../effects/runner.zig").EffectRunner;

/// Application context - passed to all view and update functions
/// Contains all necessary dependencies without global state
pub const AppContext = struct {
    /// Frame allocator - reset after each render
    allocator: std.mem.Allocator,
    
    /// Frame arena - owns the allocator
    frame_arena: *std.heap.ArenaAllocator,
    
    /// Application theme - immutable during frame
    theme: *const Theme,
    
    /// Router instance - for navigation
    router: *Router,
    
    /// Effect runner - for scheduling effects
    effects: *EffectRunner,
    
    /// String interner - for deduplication
    interner: *StringInterner,
    
    /// Frame number - for debugging
    frame_number: u64,
    
    /// Create a new application context
    pub fn init(
        frame_arena: *std.heap.ArenaAllocator,
        theme: *const Theme,
        router: *Router,
        effects: *EffectRunner,
        interner: *StringInterner,
    ) AppContext {
        return .{
            .allocator = frame_arena.allocator(),
            .frame_arena = frame_arena,
            .theme = theme,
            .router = router,
            .effects = effects,
            .interner = interner,
            .frame_number = 0,
        };
    }
    
    /// Reset frame allocator
    pub fn resetFrame(self: *AppContext) void {
        _ = self.frame_arena.reset(.retain_capacity);
        self.frame_number += 1;
    }
    
    /// Get current route
    pub fn currentRoute(self: *const AppContext) []const u8 {
        return self.router.currentPath();
    }
    
    /// Navigate to a route
    pub fn navigate(self: *AppContext, comptime route: anytype, params: anytype) !void {
        try self.router.navigate(route, params);
    }
    
    /// Schedule an effect
    pub fn scheduleEffect(self: *AppContext, effect: anytype) !void {
        try self.effects.schedule(effect);
    }
    
    /// Intern a string
    pub fn intern(self: *AppContext, s: []const u8) ![]const u8 {
        return self.interner.intern(s);
    }
};
```

---

## PASSING CONTEXT

### En View Functions

```zig
pub fn view(model: *const Model, ctx: *AppContext) Element(Msg) {
    return h.div(ctx, .{
        .style = ctx.theme.container.flex(),
    }, &.{
        Header(ctx, model.title),
        Content(ctx, model),
        Footer(ctx),
    });
}

fn Header(ctx: *AppContext, title: []const u8) Element(Msg) {
    return h.header(ctx, .{
        .style = ctx.theme.header.default(),
    }, &.{
        h.h1(ctx, .{}, &.{
            h.text(ctx, title),
        }),
    });
}
```

### En Update Functions

```zig
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .navigate_home => {
            ctx.navigate(routes.home, .{}) catch {};
            return Effect.none;
        },
        
        .fetch_data => {
            return Effect.http(.{
                .url = "/api/data",
                .on_success = .data_loaded,
                .on_error = .fetch_failed,
            });
        },
        
        .increment => {
            model.count += 1;
            return Effect.none;
        },
    };
}
```

### En Components

```zig
/// Reusable component
pub fn Button(
    ctx: *AppContext,
    comptime Msg: type,
    config: struct {
        text: []const u8,
        onClick: Msg,
        variant: enum { primary, secondary } = .primary,
    },
) Element(Msg) {
    const style = switch (config.variant) {
        .primary => ctx.theme.button.primary(),
        .secondary => ctx.theme.button.secondary(),
    };
    
    return h.button(ctx, .{
        .onClick = config.onClick,
        .style = style,
    }, &.{
        h.text(ctx, config.text),
    });
}

// Usage
Button(ctx, Msg, .{
    .text = "Click me",
    .onClick = .button_clicked,
    .variant = .primary,
});
```

---

## CONTEXT BUILDERS

### Test Context

```zig
/// Context for testing
pub const TestContext = struct {
    memory: MemoryStrategy,
    theme: Theme,
    router: Router,
    effects: EffectRunner,
    interner: StringInterner,
    ctx: AppContext,
    
    pub fn init() !TestContext {
        var memory = try MemoryStrategy.init();
        errdefer memory.deinit();
        
        var theme = Theme.default();
        var router = try Router.init(memory.persistent());
        errdefer router.deinit();
        
        var effects = try EffectRunner.init(memory.persistent());
        errdefer effects.deinit();
        
        var interner = StringInterner.init(memory.persistent());
        errdefer interner.deinit();
        
        const ctx = AppContext.init(
            &memory.frame_arena,
            &theme,
            &router,
            &effects,
            &interner,
        );
        
        return .{
            .memory = memory,
            .theme = theme,
            .router = router,
            .effects = effects,
            .interner = interner,
            .ctx = ctx,
        };
    }
    
    pub fn deinit(self: *TestContext) void {
        self.interner.deinit();
        self.effects.deinit();
        self.router.deinit();
        self.memory.deinit();
    }
    
    pub fn reset(self: *TestContext) void {
        self.ctx.resetFrame();
    }
};
```

### Production Context

```zig
/// Context for production
pub const ProductionContext = struct {
    memory: MemoryStrategy,
    theme: Theme,
    router: Router,
    effects: EffectRunner,
    interner: StringInterner,
    ctx: AppContext,
    
    pub fn init(config: struct {
        theme: ?Theme = null,
        initial_route: ?[]const u8 = null,
    }) !ProductionContext {
        var memory = try MemoryStrategy.init();
        errdefer memory.deinit();
        
        var theme = config.theme orelse Theme.default();
        
        var router = try Router.init(memory.persistent());
        errdefer router.deinit();
        
        if (config.initial_route) |route| {
            try router.setInitialPath(route);
        }
        
        var effects = try EffectRunner.init(memory.persistent());
        errdefer effects.deinit();
        
        var interner = StringInterner.init(memory.persistent());
        errdefer interner.deinit();
        
        const ctx = AppContext.init(
            &memory.frame_arena,
            &theme,
            &router,
            &effects,
            &interner,
        );
        
        return .{
            .memory = memory,
            .theme = theme,
            .router = router,
            .effects = effects,
            .interner = interner,
            .ctx = ctx,
        };
    }
    
    pub fn deinit(self: *ProductionContext) void {
        self.interner.deinit();
        self.effects.deinit();
        self.router.deinit();
        self.memory.deinit();
    }
};
```

---

## CONTEXT EXTENSION

### Custom Context Fields

```zig
/// Extended context for specific app needs
pub fn ExtendedContext(comptime Extra: type) type {
    return struct {
        base: AppContext,
        extra: Extra,
        
        pub fn init(base: AppContext, extra: Extra) @This() {
            return .{
                .base = base,
                .extra = extra,
            };
        }
        
        // Forward base methods
        pub fn resetFrame(self: *@This()) void {
            self.base.resetFrame();
        }
        
        pub fn currentRoute(self: *const @This()) []const u8 {
            return self.base.currentRoute();
        }
        
        pub fn navigate(self: *@This(), comptime route: anytype, params: anytype) !void {
            try self.base.navigate(route, params);
        }
    };
}

// Usage
const MyExtra = struct {
    user_id: u32,
    session_token: []const u8,
};

const MyContext = ExtendedContext(MyExtra);

pub fn view(model: *const Model, ctx: *MyContext) Element(Msg) {
    // Access base context
    const element = h.div(&ctx.base, .{}, &.{});
    
    // Access extra fields
    const user_id = ctx.extra.user_id;
    
    return element;
}
```

---

## CONTEXT SCOPING

### Nested Contexts

```zig
/// Create a nested context with modified theme
pub fn withTheme(parent: *AppContext, theme: *const Theme) AppContext {
    return .{
        .allocator = parent.allocator,
        .frame_arena = parent.frame_arena,
        .theme = theme, // Override theme
        .router = parent.router,
        .effects = parent.effects,
        .interner = parent.interner,
        .frame_number = parent.frame_number,
    };
}

// Usage
pub fn ThemedSection(ctx: *AppContext, theme: *const Theme) Element(Msg) {
    var scoped_ctx = withTheme(ctx, theme);
    
    return h.section(&scoped_ctx, .{
        .style = scoped_ctx.theme.section.default(),
    }, &.{
        // Child components use scoped theme
        Header(&scoped_ctx),
        Content(&scoped_ctx),
    });
}
```

---

## TESTING WITH CONTEXT

### Unit Tests

```zig
test "context in view" {
    var test_ctx = try TestContext.init();
    defer test_ctx.deinit();
    
    const model = Model{ .count = 0 };
    const element = view(&model, &test_ctx.ctx);
    
    try testing.expect(element == .element);
}

test "context in update" {
    var test_ctx = try TestContext.init();
    defer test_ctx.deinit();
    
    var model = Model{ .count = 0 };
    const effect = update(&model, .increment, &test_ctx.ctx);
    
    try testing.expectEqual(@as(i32, 1), model.count);
    try testing.expect(effect == .none);
}
```

### Integration Tests

```zig
test "full app with context" {
    var test_ctx = try TestContext.init();
    defer test_ctx.deinit();
    
    // Initialize app
    var app = try App.init(test_ctx.memory.persistent());
    defer app.deinit();
    
    // Render
    const element = app.view(&test_ctx.ctx);
    
    // Update
    const effect = app.update(.some_action, &test_ctx.ctx);
    
    // Assert
    try testing.expect(element == .element);
    try testing.expect(effect == .none);
}
```

---

## PERFORMANCE CONSIDERATIONS

### 1. Context Size
- AppContext is ~64 bytes (8 pointers)
- Passing by pointer is efficient
- No copy overhead

### 2. Frame Arena
- Reset is O(1) with `.retain_capacity`
- No memory fragmentation
- Ideal for hot paths

### 3. String Interning
- Hash once, compare by pointer
- Reduces duplicate allocations
- Trade-off: initial overhead vs later savings

---

## ANTI-PATTERNS

### ❌ Store Context in Model

```zig
// BAD: Don't store context in state
pub const Model = struct {
    ctx: *AppContext, // ❌ NO!
    count: i32,
};
```

### ❌ Mutate Context

```zig
// BAD: Context should be treated as immutable in view
pub fn view(model: *const Model, ctx: *AppContext) Element(Msg) {
    ctx.frame_number = 999; // ❌ NO!
    return h.div(ctx, .{}, &.{});
}
```

### ❌ Store Frame Arena Allocations

```zig
// BAD: Don't store allocations from frame arena
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    const data = try ctx.allocator.alloc(u8, 100);
    model.persistent_data = data; // ❌ Will be freed on resetFrame!
    return Effect.none;
}
```

---

## BEST PRACTICES

### ✅ Always Pass Context

```zig
// Correct: Explicit context in all functions
pub fn view(model: *const Model, ctx: *AppContext) Element(Msg) {
    return Container(ctx, model);
}

fn Container(ctx: *AppContext, model: *const Model) Element(Msg) {
    return h.div(ctx, .{}, &.{
        Header(ctx),
        Body(ctx, model),
    });
}
```

### ✅ Use Context Builders for Tests

```zig
test "with test context" {
    var test_ctx = try TestContext.init();
    defer test_ctx.deinit();

    // Test with all dependencies configured
    const result = myFunction(&test_ctx.ctx);
    try testing.expect(result);
}
```

### ✅ Document Context Requirements

```zig
/// Renders a user profile
///
/// Requires:
/// - ctx.theme for styling
/// - ctx.router for navigation links
pub fn UserProfile(ctx: *AppContext, user: User) Element(Msg) {
    // ...
}
```

---

## CONCLUSION

The context system eliminates global state, improves testability, and makes data flow explicit and predictable. It is fundamental to the ZUI v6 architecture.

**Links:**
- [← Previous: Core Infrastructure](01-core-infrastructure.md)
- [→ Next: Virtual DOM](03-virtual-dom.md)
- [↑ Back to index](../README.md)
