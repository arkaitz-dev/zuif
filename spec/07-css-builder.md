# 07 - CSS BUILDER

**Document:** Atomic CSS Builder
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

ZUI's CSS Builder automatically generates atomic classes from StyleRule, enabling efficient reuse and a minimal CSS bundle.

---

## ATOMIC CSS GENERATION

### CssBuilder Structure

```zig
// src/css/builder.zig
const std = @import("std");
const StyleRule = @import("rules.zig").StyleRule;

pub const CssBuilder = struct {
    allocator: std.mem.Allocator,
    class_map: std.StringHashMap(ClassInfo),
    next_id: u32,

    const ClassInfo = struct {
        class_name: []const u8,
        css_rule: []const u8,
        usage_count: u32,
    };

    pub fn init(allocator: std.mem.Allocator) CssBuilder {
        return .{
            .allocator = allocator,
            .class_map = std.StringHashMap(ClassInfo).init(allocator),
            .next_id = 0,
        };
    }

    pub fn deinit(self: *CssBuilder) void {
        var iter = self.class_map.iterator();
        while (iter.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.class_name);
            self.allocator.free(entry.value_ptr.css_rule);
        }
        self.class_map.deinit();
    }

    /// Generate atomic classes for a style rule
    pub fn build(self: *CssBuilder, rule: StyleRule) ![]const u8 {
        var classes = std.ArrayList([]const u8).init(self.allocator);
        defer classes.deinit();

        // Generate one class per property
        inline for (std.meta.fields(StyleRule)) |field| {
            const value = @field(rule, field.name);
            if (value != null) {
                const class_name = try self.getOrCreateClass(field.name, value);
                try classes.append(class_name);
            }
        }

        // Join class names with spaces
        return try std.mem.join(self.allocator, " ", classes.items);
    }

    /// Get or create atomic class for a property
    fn getOrCreateClass(
        self: *CssBuilder,
        property: []const u8,
        value: anytype,
    ) ![]const u8 {
        // Create a unique key for this property-value pair
        const key = try self.makeKey(property, value);
        defer self.allocator.free(key);

        // Check if class already exists
        if (self.class_map.get(key)) |info| {
            // Increment usage count
            var mutable_info = info;
            mutable_info.usage_count += 1;
            try self.class_map.put(key, mutable_info);
            return info.class_name;
        }

        // Create new class
        const class_name = try self.generateClassName(property);
        const css_rule = try self.generateCssRule(property, value, class_name);

        const owned_key = try self.allocator.dupe(u8, key);
        const info = ClassInfo{
            .class_name = class_name,
            .css_rule = css_rule,
            .usage_count = 1,
        };

        try self.class_map.put(owned_key, info);
        return class_name;
    }

    /// Generate unique class name
    fn generateClassName(self: *CssBuilder, property: []const u8) ![]const u8 {
        const id = self.next_id;
        self.next_id += 1;

        return try std.fmt.allocPrint(
            self.allocator,
            "zui-{s}-{d}",
            .{ property, id },
        );
    }

    /// Generate CSS rule string
    fn generateCssRule(
        self: *CssBuilder,
        property: []const u8,
        value: anytype,
        class_name: []const u8,
    ) ![]const u8 {
        const css_property = try self.toCssProperty(property);
        const css_value = try self.toCssValue(value);

        return try std.fmt.allocPrint(
            self.allocator,
            ".{s} {{ {s}: {s}; }}",
            .{ class_name, css_property, css_value },
        );
    }

    /// Convert Zig property name to CSS property name
    fn toCssProperty(self: *CssBuilder, property: []const u8) ![]const u8 {
        _ = self;

        // Convert snake_case to kebab-case
        var result = std.ArrayList(u8).init(self.allocator);
        defer result.deinit();

        for (property) |c| {
            if (c == '_') {
                try result.append('-');
            } else {
                try result.append(c);
            }
        }

        return result.toOwnedSlice();
    }

    /// Convert value to CSS string
    fn toCssValue(self: *CssBuilder, value: anytype) ![]const u8 {
        _ = self;
        const T = @TypeOf(value);

        return switch (@typeInfo(T)) {
            .Enum => @tagName(value),
            .Int => try std.fmt.allocPrint(self.allocator, "{d}", .{value}),
            .Float => try std.fmt.allocPrint(self.allocator, "{d}", .{value}),
            .Pointer => |ptr_info| {
                if (ptr_info.child == u8) {
                    return value; // String
                }
                @compileError("Unsupported pointer type");
            },
            .Struct => {
                // Assume it has a toCss() method
                return value.toCss(self.allocator);
            },
            else => @compileError("Unsupported value type for CSS"),
        };
    }

    /// Create unique key for property-value pair
    fn makeKey(self: *CssBuilder, property: []const u8, value: anytype) ![]const u8 {
        const value_str = try self.toCssValue(value);
        defer self.allocator.free(value_str);

        return try std.fmt.allocPrint(
            self.allocator,
            "{s}:{s}",
            .{ property, value_str },
        );
    }

    /// Export all CSS rules to string
    pub fn exportCss(self: *CssBuilder) ![]const u8 {
        var result = std.ArrayList(u8).init(self.allocator);
        defer result.deinit();

        var writer = result.writer();

        // Write header
        try writer.writeAll("/* Generated by ZUI CSS Builder */\n\n");

        // Sort classes by usage count (most used first)
        var entries = std.ArrayList(ClassInfo).init(self.allocator);
        defer entries.deinit();

        var iter = self.class_map.valueIterator();
        while (iter.next()) |info| {
            try entries.append(info.*);
        }

        std.sort.sort(ClassInfo, entries.items, {}, struct {
            fn lessThan(_: void, a: ClassInfo, b: ClassInfo) bool {
                return a.usage_count > b.usage_count;
            }
        }.lessThan);

        // Write CSS rules
        for (entries.items) |info| {
            try writer.print("{s}\n", .{info.css_rule});
        }

        return result.toOwnedSlice();
    }

    /// Get statistics
    pub fn getStats(self: *CssBuilder) Stats {
        var total_usage: u32 = 0;
        var iter = self.class_map.valueIterator();
        while (iter.next()) |info| {
            total_usage += info.usage_count;
        }

        return .{
            .total_classes = @intCast(self.class_map.count()),
            .total_usage = total_usage,
            .avg_reuse = if (self.class_map.count() > 0)
                @as(f32, @floatFromInt(total_usage)) / @as(f32, @floatFromInt(self.class_map.count()))
            else
                0,
        };
    }

    pub const Stats = struct {
        total_classes: u32,
        total_usage: u32,
        avg_reuse: f32,
    };
};
```

---

## STYLES NAMESPACE

### High-Level API

```zig
// src/css/styles.zig
const std = @import("std");
const StyleRule = @import("rules.zig").StyleRule;
const CssBuilder = @import("builder.zig").CssBuilder;

/// Global CSS builder instance (per context)
var builder: ?*CssBuilder = null;

/// Initialize styles system
pub fn init(allocator: std.mem.Allocator) !void {
    builder = try allocator.create(CssBuilder);
    builder.?.* = CssBuilder.init(allocator);
}

/// Cleanup styles system
pub fn deinit(allocator: std.mem.Allocator) void {
    if (builder) |b| {
        b.deinit();
        allocator.destroy(b);
        builder = null;
    }
}

/// Create style classes from rule
pub fn create(rule: StyleRule) []const u8 {
    if (builder) |b| {
        return b.build(rule) catch "";
    }
    return "";
}

/// Export CSS
pub fn exportCss(allocator: std.mem.Allocator) ![]const u8 {
    if (builder) |b| {
        return b.exportCss();
    }
    return try allocator.dupe(u8, "");
}

/// Preset styles for common patterns
pub const container = struct {
    pub fn flex() StyleRule {
        return .{
            .display = .flex,
            .flex_direction = .column,
        };
    }

    pub fn flexRow() StyleRule {
        return .{
            .display = .flex,
            .flex_direction = .row,
        };
    }

    pub fn centered() StyleRule {
        return .{
            .display = .flex,
            .justify_content = .center,
            .align_items = .center,
        };
    }

    pub fn fullScreen() StyleRule {
        return .{
            .width = .{ .percent = 100 },
            .height = .{ .vh = 100 },
        };
    }
};

pub const button = struct {
    pub fn primary() StyleRule {
        return .{
            .padding = .{ .rem = 0.75 },
            .background_color = .{ .hex = 0x3b82f6 },
            .color = .{ .hex = 0xffffff },
            .border_radius = .{ .px = 8 },
            .cursor = .pointer,
            .font_weight = .{ .number = 600 },
        };
    }

    pub fn secondary() StyleRule {
        return .{
            .padding = .{ .rem = 0.75 },
            .background_color = .{ .hex = 0x6b7280 },
            .color = .{ .hex = 0xffffff },
            .border_radius = .{ .px = 8 },
            .cursor = .pointer,
            .font_weight = .{ .number = 600 },
        };
    }

    pub fn danger() StyleRule {
        return .{
            .padding = .{ .rem = 0.75 },
            .background_color = .{ .hex = 0xef4444 },
            .color = .{ .hex = 0xffffff },
            .border_radius = .{ .px = 8 },
            .cursor = .pointer,
            .font_weight = .{ .number = 600 },
        };
    }
};

pub const text = struct {
    pub fn heading() StyleRule {
        return .{
            .font_size = .{ .rem = 2 },
            .font_weight = .{ .number = 700 },
            .line_height = .{ .rem = 2.5 },
        };
    }

    pub fn body() StyleRule {
        return .{
            .font_size = .{ .rem = 1 },
            .line_height = .{ .rem = 1.5 },
        };
    }

    pub fn small() StyleRule {
        return .{
            .font_size = .{ .rem = 0.875 },
            .line_height = .{ .rem = 1.25 },
        };
    }
};

pub const card = struct {
    pub fn default() StyleRule {
        return .{
            .background_color = .{ .hex = 0xffffff },
            .border_width = .{ .px = 1 },
            .border_color = .{ .hex = 0xe5e7eb },
            .border_style = .solid,
            .border_radius = .{ .px = 12 },
            .padding = .{ .rem = 1.5 },
        };
    }

    pub fn elevated() StyleRule {
        return .{
            .background_color = .{ .hex = 0xffffff },
            .border_radius = .{ .px = 12 },
            .padding = .{ .rem = 1.5 },
            // Shadow would be added via transform/box-shadow
        };
    }
};
```

---

## USAGE EXAMPLES

### Creating Styles

```zig
const styles = @import("css/styles.zig");

pub fn view(model: *const Model, ctx: *AppContext) Element(Msg) {
    return h.div(ctx, .{
        .class = styles.create(.{
            .display = .flex,
            .flex_direction = .column,
            .gap = .{ .rem = 1 },
            .padding = .{ .rem = 2 },
        }),
    }, &.{
        h.h1(ctx, .{
            .class = styles.create(styles.text.heading()),
        }, &.{
            h.text(ctx, "Welcome"),
        }),

        h.button(ctx, .{
            .class = styles.create(styles.button.primary()),
            .onClick = .clicked,
        }, &.{
            h.text(ctx, "Click me"),
        }),
    });
}
```

### Style Composition

```zig
const base_style = styles.button.primary();
const custom_style = StyleRule{
    .font_size = .{ .rem = 1.2 },
    .padding = .{ .rem = 1 },
};

const merged = base_style.merge(custom_style);
const class_name = styles.create(merged);
```

### Usage Statistics

```zig
const stats = builder.getStats();
std.debug.print("Total classes: {d}\n", .{stats.total_classes});
std.debug.print("Average reuse: {d:.2}\n", .{stats.avg_reuse});
```

---

## OPTIMIZATIONS

### 1. Automatic Deduplication
- Same property-value → Same class
- Reduces final CSS size

### 2. Class Name Caching
- StringHashMap for O(1) lookup
- Avoids regeneration

### 3. Usage Tracking
- Reuse statistics
- Usage-based optimization

### 4. Atomic Output
- One property per class
- Maximum reusability

---

## CONCLUSION

The CSS Builder provides automatic generation of atomic classes with deduplication, resulting in minimal and highly reusable CSS.

**Links:**
- [← Previous: CSS Rules](06-css-rules.md)
- [→ Next: CSS Extraction](08-css-extraction.md)
- [↑ Back to index](../README.md)
