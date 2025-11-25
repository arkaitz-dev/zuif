# 01 - CORE INFRASTRUCTURE

**Document:** Core Infrastructure
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

The ZUI core infrastructure provides fundamental building blocks for memory management, string utilities, and basic data structures. Everything is designed for the `wasm32-freestanding` target.

---

## MEMORY MANAGEMENT

### Allocator Strategy

ZUI uses two main allocation strategies:

```zig
// src/core/memory.zig
const std = @import("std");

/// Memory strategy for ZUI applications
pub const MemoryStrategy = struct {
    /// General Purpose Allocator for persistent data
    gpa: std.heap.GeneralPurposeAllocator(.{}),
    
    /// Arena for per-frame allocations
    frame_arena: std.heap.ArenaAllocator,
    
    pub fn init() !MemoryStrategy {
        var gpa = std.heap.GeneralPurposeAllocator(.{}){};
        const gpa_allocator = gpa.allocator();
        
        return .{
            .gpa = gpa,
            .frame_arena = std.heap.ArenaAllocator.init(gpa_allocator),
        };
    }
    
    pub fn deinit(self: *MemoryStrategy) void {
        self.frame_arena.deinit();
        _ = self.gpa.deinit();
    }
    
    /// Reset frame arena after each render
    pub fn resetFrame(self: *MemoryStrategy) void {
        _ = self.frame_arena.reset(.retain_capacity);
    }
    
    /// Get allocator for persistent data
    pub fn persistent(self: *MemoryStrategy) std.mem.Allocator {
        return self.gpa.allocator();
    }
    
    /// Get allocator for frame-scoped data
    pub fn frame(self: *MemoryStrategy) std.mem.Allocator {
        return self.frame_arena.allocator();
    }
};
```

### Usage Rules

#### 1. Persistent Data (GPA)
```zig
// Model state - persists between frames
pub const Model = struct {
    users: std.ArrayList(User),
    
    pub fn init(allocator: std.mem.Allocator) !Model {
        return .{
            .users = std.ArrayList(User).init(allocator),
        };
    }
    
    pub fn deinit(self: *Model) void {
        self.users.deinit();
    }
};
```

#### 2. Temporary Data (Arena)
```zig
// Virtual DOM - lives only during the frame
pub fn view(model: *const Model, ctx: *AppContext) Element(Msg) {
    // ctx.allocator is the frame arena
    const items = ctx.allocator.alloc(Element(Msg), model.users.items.len);
    // No deinit needed - arena cleans up automatically
    return h.div(ctx, .{}, items);
}
```

### Memory Safety Checks

```zig
/// Wrapper for debugging memory issues
pub const DebugAllocator = struct {
    child: std.mem.Allocator,
    allocations: std.AutoHashMap(usize, AllocationInfo),
    
    const AllocationInfo = struct {
        size: usize,
        alignment: u29,
        stack_trace: ?*std.builtin.StackTrace,
    };
    
    pub fn init(child: std.mem.Allocator) DebugAllocator {
        return .{
            .child = child,
            .allocations = std.AutoHashMap(usize, AllocationInfo).init(child),
        };
    }
    
    pub fn allocator(self: *DebugAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = resize,
                .free = free,
            },
        };
    }
    
    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u29, ret_addr: usize) ?[*]u8 {
        const self = @ptrCast(*DebugAllocator, @alignCast(@alignOf(DebugAllocator), ctx));
        const result = self.child.rawAlloc(len, ptr_align, ret_addr);
        
        if (result) |ptr| {
            self.allocations.put(@ptrToInt(ptr), .{
                .size = len,
                .alignment = ptr_align,
                .stack_trace = if (std.builtin.mode == .Debug) 
                    std.builtin.default_stack_trace 
                else 
                    null,
            }) catch {};
        }
        
        return result;
    }
    
    fn free(ctx: *anyopaque, buf: []u8, ptr_align: u29, ret_addr: usize) void {
        const self = @ptrCast(*DebugAllocator, @alignCast(@alignOf(DebugAllocator), ctx));
        _ = self.allocations.remove(@ptrToInt(buf.ptr));
        self.child.rawFree(buf, ptr_align, ret_addr);
    }
    
    fn resize(ctx: *anyopaque, buf: []u8, ptr_align: u29, new_len: usize, ret_addr: usize) bool {
        const self = @ptrCast(*DebugAllocator, @alignCast(@alignOf(DebugAllocator), ctx));
        const result = self.child.rawResize(buf, ptr_align, new_len, ret_addr);
        
        if (result) {
            if (self.allocations.getPtr(@ptrToInt(buf.ptr))) |info| {
                info.size = new_len;
            }
        }
        
        return result;
    }
    
    /// Report memory leaks
    pub fn checkLeaks(self: *DebugAllocator) !void {
        var iter = self.allocations.iterator();
        var has_leaks = false;

        while (iter.next()) |entry| {
            std.debug.print("Memory leak: {d} bytes at 0x{x}\n", .{
                entry.value_ptr.size,
                entry.key_ptr.*,
            });
            has_leaks = true;
        }

        if (has_leaks) {
            return error.MemoryLeaks;
        }
    }
};
```

---

## STRING UTILITIES

### String Interning

To optimize memory and comparisons:

```zig
// src/core/string_utils.zig
const std = @import("std");

/// String interning for deduplication
pub const StringInterner = struct {
    allocator: std.mem.Allocator,
    strings: std.StringHashMap(void),
    
    pub fn init(allocator: std.mem.Allocator) StringInterner {
        return .{
            .allocator = allocator,
            .strings = std.StringHashMap(void).init(allocator),
        };
    }
    
    pub fn deinit(self: *StringInterner) void {
        var iter = self.strings.keyIterator();
        while (iter.next()) |key| {
            self.allocator.free(key.*);
        }
        self.strings.deinit();
    }
    
    /// Intern a string - returns owned slice
    pub fn intern(self: *StringInterner, s: []const u8) ![]const u8 {
        const result = try self.strings.getOrPut(s);
        
        if (!result.found_existing) {
            // Duplicate the string
            const owned = try self.allocator.dupe(u8, s);
            result.key_ptr.* = owned;
        }
        
        return result.key_ptr.*;
    }
    
    /// Check if string is interned
    pub fn contains(self: *const StringInterner, s: []const u8) bool {
        return self.strings.contains(s);
    }
};
```

### String Builder

```zig
/// Efficient string building
pub const StringBuilder = struct {
    buffer: std.ArrayList(u8),
    
    pub fn init(allocator: std.mem.Allocator) StringBuilder {
        return .{
            .buffer = std.ArrayList(u8).init(allocator),
        };
    }
    
    pub fn deinit(self: *StringBuilder) void {
        self.buffer.deinit();
    }
    
    pub fn append(self: *StringBuilder, s: []const u8) !void {
        try self.buffer.appendSlice(s);
    }
    
    pub fn appendFmt(self: *StringBuilder, comptime fmt: []const u8, args: anytype) !void {
        try self.buffer.writer().print(fmt, args);
    }
    
    pub fn clear(self: *StringBuilder) void {
        self.buffer.clearRetainingCapacity();
    }
    
    pub fn toOwnedSlice(self: *StringBuilder) ![]u8 {
        return self.buffer.toOwnedSlice();
    }
    
    pub fn items(self: *const StringBuilder) []const u8 {
        return self.buffer.items;
    }
};
```

### String Formatting

```zig
/// Format string with allocator
pub fn format(allocator: std.mem.Allocator, comptime fmt: []const u8, args: anytype) ![]u8 {
    var list = std.ArrayList(u8).init(allocator);
    defer list.deinit();
    
    try list.writer().print(fmt, args);
    return list.toOwnedSlice();
}

/// Format string into fixed buffer
pub fn formatBuf(buf: []u8, comptime fmt: []const u8, args: anytype) ![]u8 {
    return std.fmt.bufPrint(buf, fmt, args);
}
```

---

## DATA STRUCTURES

### Tagged Union Utilities

```zig
/// Helper for tagged unions
pub fn TaggedUnion(comptime T: type) type {
    return struct {
        /// Get active tag
        pub fn activeTag(value: T) std.meta.Tag(T) {
            return std.meta.activeTag(value);
        }
        
        /// Check if value matches tag
        pub fn is(value: T, comptime tag: std.meta.Tag(T)) bool {
            return activeTag(value) == tag;
        }
        
        /// Map over tagged union
        pub fn map(
            value: T,
            comptime ReturnType: type,
            mapper: anytype,
        ) ReturnType {
            inline for (std.meta.fields(T)) |field| {
                if (@field(std.meta.Tag(T), field.name) == activeTag(value)) {
                    return @call(.always_inline, mapper, .{
                        @field(value, field.name),
                    });
                }
            }
            unreachable;
        }
    };
}
```

### Result Type

```zig
/// Result type for error handling
pub fn Result(comptime T: type, comptime E: type) type {
    return union(enum) {
        ok: T,
        err: E,
        
        pub fn isOk(self: @This()) bool {
            return self == .ok;
        }
        
        pub fn isErr(self: @This()) bool {
            return self == .err;
        }
        
        pub fn unwrap(self: @This()) T {
            return switch (self) {
                .ok => |v| v,
                .err => @panic("called unwrap on error"),
            };
        }
        
        pub fn unwrapOr(self: @This(), default: T) T {
            return switch (self) {
                .ok => |v| v,
                .err => default,
            };
        }
        
        pub fn map(
            self: @This(),
            comptime U: type,
            f: fn (T) U,
        ) Result(U, E) {
            return switch (self) {
                .ok => |v| .{ .ok = f(v) },
                .err => |e| .{ .err = e },
            };
        }
    };
}
```

### Option Type

```zig
/// Option type for nullable values
pub fn Option(comptime T: type) type {
    return union(enum) {
        some: T,
        none,
        
        pub fn isSome(self: @This()) bool {
            return self == .some;
        }
        
        pub fn isNone(self: @This()) bool {
            return self == .none;
        }
        
        pub fn unwrap(self: @This()) T {
            return switch (self) {
                .some => |v| v,
                .none => @panic("called unwrap on none"),
            };
        }
        
        pub fn unwrapOr(self: @This(), default: T) T {
            return switch (self) {
                .some => |v| v,
                .none => default,
            };
        }
        
        pub fn map(
            self: @This(),
            comptime U: type,
            f: fn (T) U,
        ) Option(U) {
            return switch (self) {
                .some => |v| .{ .some = f(v) },
                .none => .none,
            };
        }
    };
}
```

---

## ERROR HANDLING

### Error Sets

```zig
/// Core errors
pub const CoreError = error{
    OutOfMemory,
    InvalidUtf8,
    BufferTooSmall,
    InvalidArgument,
};

/// Allocation errors
pub const AllocationError = error{
    OutOfMemory,
};
```

### Error Context

```zig
/// Error with context
pub fn ErrorWithContext(comptime E: type) type {
    return struct {
        err: E,
        context: []const u8,
        
        pub fn init(err: E, context: []const u8) @This() {
            return .{
                .err = err,
                .context = context,
            };
        }
        
        pub fn format(
            self: @This(),
            comptime fmt: []const u8,
            options: std.fmt.FormatOptions,
            writer: anytype,
        ) !void {
            _ = fmt;
            _ = options;
            try writer.print("{s}: {}", .{ self.context, self.err });
        }
    };
}
```

---

## TESTING UTILITIES

```zig
/// Test allocator wrapper
pub const TestAllocator = struct {
    allocator: std.mem.Allocator,
    
    pub fn init() TestAllocator {
        return .{
            .allocator = std.testing.allocator,
        };
    }
    
    pub fn alloc(self: TestAllocator, comptime T: type, n: usize) ![]T {
        return self.allocator.alloc(T, n);
    }
    
    pub fn free(self: TestAllocator, memory: anytype) void {
        self.allocator.free(memory);
    }
};

/// Expect no memory leaks in test
pub fn expectNoLeaks() !void {
    // std.testing.allocator automatically checks for leaks
}
```

---

## PERFORMANCE CONSIDERATIONS

### 1. Arena Allocations
- **Pro:** Very fast, no free overhead
- **Con:** Memory grows until reset
- **Usage:** Perfect for per-frame rendering

### 2. GPA Allocations
- **Pro:** Precise memory management
- **Con:** Tracking overhead
- **Usage:** Persistent application state

### 3. String Interning
- **Pro:** Reduces memory usage and accelerates comparisons
- **Con:** Initial hashing overhead
- **Usage:** Repeated strings (class names, tag names)

---

## USAGE EXAMPLES

### Initialization

```zig
const std = @import("std");
const core = @import("core");

pub fn main() !void {
    var memory = try core.MemoryStrategy.init();
    defer memory.deinit();
    
    // Use persistent allocator for app state
    var model = try Model.init(memory.persistent());
    defer model.deinit();
    
    // Use frame allocator for rendering
    const vdom = render(&model, memory.frame());
    defer memory.resetFrame();
}
```

### String Building

```zig
test "string builder" {
    const testing = std.testing;
    var builder = core.StringBuilder.init(testing.allocator);
    defer builder.deinit();
    
    try builder.append("Hello");
    try builder.append(" ");
    try builder.append("World");
    
    try testing.expectEqualStrings("Hello World", builder.items());
}
```

---

## CONCLUSION

The core infrastructure provides the foundation for efficient and safe memory management, optimized string utilities, and data structures that facilitate type-safe development.

**Links:**
- [← Previous: Overview](00-overview.md)
- [→ Next: Context System](02-context-system.md)
- [↑ Back to index](../README.md)
