# 06 - CSS RULES

**Document:** CSS Rules and Properties
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

This document defines the CSS rules system in ZUI, including all supported properties, their types, and how they compose into style rules.

---

## RULE STRUCTURE

### StyleRule Definition

```zig
// src/css/rules.zig
const std = @import("std");
const types = @import("types.zig");

/// Complete CSS rule with all properties
pub const StyleRule = struct {
    // Layout
    display: ?types.Display = null,
    position: ?types.Position = null,

    // Box Model
    width: ?types.Length = null,
    height: ?types.Length = null,
    min_width: ?types.Length = null,
    min_height: ?types.Length = null,
    max_width: ?types.Length = null,
    max_height: ?types.Length = null,

    // Spacing
    margin: ?types.Length = null,
    margin_top: ?types.Length = null,
    margin_right: ?types.Length = null,
    margin_bottom: ?types.Length = null,
    margin_left: ?types.Length = null,

    padding: ?types.Length = null,
    padding_top: ?types.Length = null,
    padding_right: ?types.Length = null,
    padding_bottom: ?types.Length = null,
    padding_left: ?types.Length = null,

    // Positioning
    top: ?types.Length = null,
    right: ?types.Length = null,
    bottom: ?types.Length = null,
    left: ?types.Length = null,
    z_index: ?i32 = null,

    // Flexbox
    flex_direction: ?types.FlexDirection = null,
    flex_wrap: ?types.FlexWrap = null,
    justify_content: ?types.JustifyContent = null,
    align_items: ?types.AlignItems = null,
    align_content: ?types.AlignContent = null,
    align_self: ?types.AlignSelf = null,
    flex_grow: ?f32 = null,
    flex_shrink: ?f32 = null,
    flex_basis: ?types.Length = null,
    gap: ?types.Length = null,
    row_gap: ?types.Length = null,
    column_gap: ?types.Length = null,

    // Grid
    grid_template_columns: ?types.GridTemplate = null,
    grid_template_rows: ?types.GridTemplate = null,
    grid_column: ?[]const u8 = null,
    grid_row: ?[]const u8 = null,
    grid_gap: ?types.Length = null,

    // Typography
    font_family: ?[]const u8 = null,
    font_size: ?types.Length = null,
    font_weight: ?types.FontWeight = null,
    font_style: ?types.FontStyle = null,
    line_height: ?types.Length = null,
    letter_spacing: ?types.Length = null,
    text_align: ?types.TextAlign = null,
    text_transform: ?types.TextTransform = null,
    text_decoration: ?types.TextDecoration = null,
    white_space: ?types.WhiteSpace = null,
    word_break: ?types.WordBreak = null,

    // Colors
    color: ?types.Color = null,
    background_color: ?types.Color = null,
    border_color: ?types.Color = null,

    // Borders
    border_width: ?types.Length = null,
    border_style: ?types.BorderStyle = null,
    border_radius: ?types.Length = null,
    border_top_left_radius: ?types.Length = null,
    border_top_right_radius: ?types.Length = null,
    border_bottom_left_radius: ?types.Length = null,
    border_bottom_right_radius: ?types.Length = null,

    // Visual Effects
    opacity: ?f32 = null,
    overflow: ?types.Overflow = null,
    overflow_x: ?types.Overflow = null,
    overflow_y: ?types.Overflow = null,
    visibility: ?types.Visibility = null,

    // Transforms
    transform: ?[]const u8 = null,
    transform_origin: ?[]const u8 = null,

    // Transitions
    transition: ?[]const u8 = null,
    transition_property: ?[]const u8 = null,
    transition_duration: ?u32 = null, // milliseconds
    transition_timing_function: ?types.TimingFunction = null,

    // Animations
    animation: ?[]const u8 = null,
    animation_name: ?[]const u8 = null,
    animation_duration: ?u32 = null, // milliseconds
    animation_timing_function: ?types.TimingFunction = null,
    animation_iteration_count: ?types.IterationCount = null,

    // Cursor
    cursor: ?types.Cursor = null,
    pointer_events: ?types.PointerEvents = null,

    // User Interaction
    user_select: ?types.UserSelect = null,

    /// Convert rule to CSS string
    pub fn toCss(self: StyleRule, allocator: std.mem.Allocator) ![]const u8 {
        var list = std.ArrayList(u8).init(allocator);
        var writer = list.writer();

        // Layout
        if (self.display) |val| {
            try writer.print("display: {s}; ", .{val.toCss()});
        }
        if (self.position) |val| {
            try writer.print("position: {s}; ", .{@tagName(val)});
        }

        // Box Model
        if (self.width) |val| {
            try writer.print("width: {s}; ", .{val.toCss()});
        }
        if (self.height) |val| {
            try writer.print("height: {s}; ", .{val.toCss()});
        }

        // Spacing - with shorthand optimization
        if (self.margin) |val| {
            try writer.print("margin: {s}; ", .{val.toCss()});
        } else if (self.margin_top != null or self.margin_right != null or
                   self.margin_bottom != null or self.margin_left != null) {
            if (self.margin_top) |val| {
                try writer.print("margin-top: {s}; ", .{val.toCss()});
            }
            if (self.margin_right) |val| {
                try writer.print("margin-right: {s}; ", .{val.toCss()});
            }
            if (self.margin_bottom) |val| {
                try writer.print("margin-bottom: {s}; ", .{val.toCss()});
            }
            if (self.margin_left) |val| {
                try writer.print("margin-left: {s}; ", .{val.toCss()});
            }
        }

        if (self.padding) |val| {
            try writer.print("padding: {s}; ", .{val.toCss()});
        } else if (self.padding_top != null or self.padding_right != null or
                   self.padding_bottom != null or self.padding_left != null) {
            if (self.padding_top) |val| {
                try writer.print("padding-top: {s}; ", .{val.toCss()});
            }
            if (self.padding_right) |val| {
                try writer.print("padding-right: {s}; ", .{val.toCss()});
            }
            if (self.padding_bottom) |val| {
                try writer.print("padding-bottom: {s}; ", .{val.toCss()});
            }
            if (self.padding_left) |val| {
                try writer.print("padding-left: {s}; ", .{val.toCss()});
            }
        }

        // Flexbox
        if (self.flex_direction) |val| {
            try writer.print("flex-direction: {s}; ", .{@tagName(val)});
        }
        if (self.justify_content) |val| {
            const name = @tagName(val);
            const css_name = if (std.mem.eql(u8, name, "flex_start"))
                "flex-start"
            else if (std.mem.eql(u8, name, "flex_end"))
                "flex-end"
            else if (std.mem.eql(u8, name, "space_between"))
                "space-between"
            else if (std.mem.eql(u8, name, "space_around"))
                "space-around"
            else if (std.mem.eql(u8, name, "space_evenly"))
                "space-evenly"
            else
                name;
            try writer.print("justify-content: {s}; ", .{css_name});
        }
        if (self.align_items) |val| {
            const name = @tagName(val);
            const css_name = if (std.mem.eql(u8, name, "flex_start"))
                "flex-start"
            else if (std.mem.eql(u8, name, "flex_end"))
                "flex-end"
            else
                name;
            try writer.print("align-items: {s}; ", .{css_name});
        }
        if (self.gap) |val| {
            try writer.print("gap: {s}; ", .{val.toCss()});
        }

        // Typography
        if (self.font_family) |val| {
            try writer.print("font-family: {s}; ", .{val});
        }
        if (self.font_size) |val| {
            try writer.print("font-size: {s}; ", .{val.toCss()});
        }
        if (self.font_weight) |val| {
            try writer.print("font-weight: {s}; ", .{val.toCss()});
        }
        if (self.line_height) |val| {
            try writer.print("line-height: {s}; ", .{val.toCss()});
        }
        if (self.text_align) |val| {
            try writer.print("text-align: {s}; ", .{@tagName(val)});
        }

        // Colors
        if (self.color) |val| {
            try writer.print("color: {s}; ", .{val.toCss()});
        }
        if (self.background_color) |val| {
            try writer.print("background-color: {s}; ", .{val.toCss()});
        }

        // Borders
        if (self.border_width) |val| {
            try writer.print("border-width: {s}; ", .{val.toCss()});
        }
        if (self.border_style) |val| {
            try writer.print("border-style: {s}; ", .{@tagName(val)});
        }
        if (self.border_color) |val| {
            try writer.print("border-color: {s}; ", .{val.toCss()});
        }
        if (self.border_radius) |val| {
            try writer.print("border-radius: {s}; ", .{val.toCss()});
        }

        // Visual Effects
        if (self.opacity) |val| {
            try writer.print("opacity: {d}; ", .{val});
        }
        if (self.overflow) |val| {
            try writer.print("overflow: {s}; ", .{@tagName(val)});
        }

        // Cursor
        if (self.cursor) |val| {
            try writer.print("cursor: {s}; ", .{@tagName(val)});
        }

        return list.toOwnedSlice();
    }

    /// Merge two rules, with other taking precedence
    pub fn merge(self: StyleRule, other: StyleRule) StyleRule {
        var result = self;

        inline for (std.meta.fields(StyleRule)) |field| {
            const other_val = @field(other, field.name);
            if (other_val != null) {
                @field(result, field.name) = other_val;
            }
        }

        return result;
    }

    /// Check if rule is empty
    pub fn isEmpty(self: StyleRule) bool {
        inline for (std.meta.fields(StyleRule)) |field| {
            if (@field(self, field.name) != null) {
                return false;
            }
        }
        return true;
    }
};
```

---

## ADDITIONAL TYPES

### Border Style

```zig
pub const BorderStyle = enum {
    none,
    solid,
    dashed,
    dotted,
    double,
    groove,
    ridge,
    inset,
    outset,
};
```

### Overflow

```zig
pub const Overflow = enum {
    visible,
    hidden,
    scroll,
    auto,
};
```

### Visibility

```zig
pub const Visibility = enum {
    visible,
    hidden,
    collapse,
};
```

### Cursor

```zig
pub const Cursor = enum {
    auto,
    default,
    pointer,
    text,
    move,
    not_allowed,
    grab,
    grabbing,
    crosshair,
    help,
    wait,
    progress,

    pub fn toCss(self: Cursor) []const u8 {
        return switch (self) {
            .not_allowed => "not-allowed",
            else => @tagName(self),
        };
    }
};
```

### Timing Functions

```zig
pub const TimingFunction = enum {
    linear,
    ease,
    ease_in,
    ease_out,
    ease_in_out,

    pub fn toCss(self: TimingFunction) []const u8 {
        return switch (self) {
            .ease_in => "ease-in",
            .ease_out => "ease-out",
            .ease_in_out => "ease-in-out",
            else => @tagName(self),
        };
    }
};
```

### User Select

```zig
pub const UserSelect = enum {
    none,
    auto,
    text,
    all,
};
```

### Pointer Events

```zig
pub const PointerEvents = enum {
    auto,
    none,
};
```

### White Space

```zig
pub const WhiteSpace = enum {
    normal,
    nowrap,
    pre,
    pre_wrap,
    pre_line,

    pub fn toCss(self: WhiteSpace) []const u8 {
        return switch (self) {
            .pre_wrap => "pre-wrap",
            .pre_line => "pre-line",
            else => @tagName(self),
        };
    }
};
```

### Word Break

```zig
pub const WordBreak = enum {
    normal,
    break_all,
    keep_all,
    break_word,

    pub fn toCss(self: WordBreak) []const u8 {
        return switch (self) {
            .break_all => "break-all",
            .keep_all => "keep-all",
            .break_word => "break-word",
            else => @tagName(self),
        };
    }
};
```

---

## RULE COMPOSITION

### Rule Builder

```zig
pub const RuleBuilder = struct {
    rule: StyleRule,

    pub fn init() RuleBuilder {
        return .{ .rule = .{} };
    }

    pub fn display(self: *RuleBuilder, val: types.Display) *RuleBuilder {
        self.rule.display = val;
        return self;
    }

    pub fn flexDirection(self: *RuleBuilder, val: types.FlexDirection) *RuleBuilder {
        self.rule.flex_direction = val;
        return self;
    }

    pub fn justifyContent(self: *RuleBuilder, val: types.JustifyContent) *RuleBuilder {
        self.rule.justify_content = val;
        return self;
    }

    pub fn padding(self: *RuleBuilder, val: types.Length) *RuleBuilder {
        self.rule.padding = val;
        return self;
    }

    pub fn margin(self: *RuleBuilder, val: types.Length) *RuleBuilder {
        self.rule.margin = val;
        return self;
    }

    pub fn color(self: *RuleBuilder, val: types.Color) *RuleBuilder {
        self.rule.color = val;
        return self;
    }

    pub fn backgroundColor(self: *RuleBuilder, val: types.Color) *RuleBuilder {
        self.rule.background_color = val;
        return self;
    }

    pub fn fontSize(self: *RuleBuilder, val: types.Length) *RuleBuilder {
        self.rule.font_size = val;
        return self;
    }

    pub fn build(self: *RuleBuilder) StyleRule {
        return self.rule;
    }
};
```

---

## USAGE EXAMPLES

### Basic Rule

```zig
const rule = StyleRule{
    .display = .flex,
    .flex_direction = .column,
    .gap = .{ .rem = 1 },
    .padding = .{ .rem = 2 },
};

const css = try rule.toCss(allocator);
// "display: flex; flex-direction: column; gap: 1rem; padding: 2rem;"
```

### Rule Composition

```zig
const base = StyleRule{
    .display = .flex,
    .padding = .{ .rem = 1 },
};

const override = StyleRule{
    .padding = .{ .rem = 2 },
    .gap = .{ .rem = 0.5 },
};

const merged = base.merge(override);
// display: flex, padding: 2rem (overridden), gap: 0.5rem (added)
```

### Builder Pattern

```zig
const rule = RuleBuilder.init()
    .display(.flex)
    .flexDirection(.column)
    .padding(.{ .rem = 2 })
    .gap(.{ .rem = 1 })
    .build();
```

---

## CONCLUSION

The CSS rules system provides a complete type-safe API for all common CSS properties, with composition and optimized conversion to CSS.

**Links:**
- [← Previous: CSS Types](05-css-types.md)
- [→ Next: CSS Builder](07-css-builder.md)
- [↑ Back to index](../README.md)
