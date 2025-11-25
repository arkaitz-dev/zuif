# 05 - CSS TYPES

**Document:** Type-Safe CSS Types
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

ZSS (Zig Style Sheets) provides type-safe types for all CSS values, guaranteeing validity at compile time.

---

## FUNDAMENTAL TYPES

### Length Units

```zig
// src/css/types.zig
pub const Length = union(enum) {
    px: f32,
    rem: f32,
    em: f32,
    percent: f32,
    vh: f32,
    vw: f32,
    auto,
    zero,

    pub fn toCss(self: Length) []const u8 {
        return switch (self) {
            .px => |v| std.fmt.allocPrint("{d}px", .{v}),
            .rem => |v| std.fmt.allocPrint("{d}rem", .{v}),
            .em => |v| std.fmt.allocPrint("{d}em", .{v}),
            .percent => |v| std.fmt.allocPrint("{d}%", .{v}),
            .vh => |v| std.fmt.allocPrint("{d}vh", .{v}),
            .vw => |v| std.fmt.allocPrint("{d}vw", .{v}),
            .auto => "auto",
            .zero => "0",
        };
    }
};
```

### Colors

```zig
pub const Color = union(enum) {
    hex: u24,
    rgb: struct { r: u8, g: u8, b: u8 },
    rgba: struct { r: u8, g: u8, b: u8, a: f32 },
    hsl: struct { h: u16, s: u8, l: u8 },
    hsla: struct { h: u16, s: u8, l: u8, a: f32 },
    named: []const u8,

    pub fn toCss(self: Color) []const u8 {
        return switch (self) {
            .hex => |v| std.fmt.allocPrint("#{x:0>6}", .{v}),
            .rgb => |v| std.fmt.allocPrint("rgb({d},{d},{d})", .{v.r, v.g, v.b}),
            .rgba => |v| std.fmt.allocPrint("rgba({d},{d},{d},{d})", .{v.r, v.g, v.b, v.a}),
            .named => |n| n,
        };
    }
};
```

### Display

```zig
pub const Display = enum {
    block,
    inline,
    inline_block,
    flex,
    inline_flex,
    grid,
    inline_grid,
    none,

    pub fn toCss(self: Display) []const u8 {
        return switch (self) {
            .inline_block => "inline-block",
            .inline_flex => "inline-flex",
            .inline_grid => "inline-grid",
            else => @tagName(self),
        };
    }
};
```

### Flexbox

```zig
pub const FlexDirection = enum { row, row_reverse, column, column_reverse };
pub const JustifyContent = enum { flex_start, flex_end, center, space_between, space_around, space_evenly };
pub const AlignItems = enum { flex_start, flex_end, center, stretch, baseline };
pub const FlexWrap = enum { nowrap, wrap, wrap_reverse };
```

### Grid

```zig
pub const GridTemplate = union(enum) {
    tracks: []const []const u8,
    repeat: struct {
        count: u32,
        track: []const u8,
    },
};
```

### Position

```zig
pub const Position = enum { static, relative, absolute, fixed, sticky };
```

### Typography

```zig
pub const FontWeight = union(enum) {
    number: u16,
    normal,
    bold,
    bolder,
    lighter,
};

pub const FontStyle = enum { normal, italic, oblique };
pub const TextAlign = enum { left, right, center, justify };
pub const TextTransform = enum { none, uppercase, lowercase, capitalize };
pub const TextDecoration = enum { none, underline, overline, line_through };
```

---

## HELPERS

```zig
pub fn spacing(value: anytype) Length {
    return switch (@TypeOf(value)) {
        comptime_int, comptime_float => .{ .rem = @floatFromInt(value) },
        f32 => .{ .rem = value },
        else => @compileError("Invalid spacing type"),
    };
}

pub fn color(hex: u24) Color {
    return .{ .hex = hex };
}
```

**Links:**
- [← Previous: HTML DSL](04-html-dsl.md)
- [→ Next: CSS Rules](06-css-rules.md)
- [↑ Back to index](../README.md)
