# 09 - CSS THEME

**Document:** Theme System
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

ZUI's theme system provides a consistent set of design tokens (colors, spacing, typography) that can be referenced throughout the application, with support for dark mode and custom themes.

---

## THEME DEFINITION

### Core Theme Structure

```zig
// src/css/theme.zig
const std = @import("std");
const types = @import("types.zig");

pub const Theme = struct {
    colors: ColorPalette,
    spacing: SpacingScale,
    typography: Typography,
    borders: Borders,
    shadows: Shadows,
    transitions: Transitions,

    /// Default light theme
    pub fn default() Theme {
        return .{
            .colors = ColorPalette.light(),
            .spacing = SpacingScale.default(),
            .typography = Typography.default(),
            .borders = Borders.default(),
            .shadows = Shadows.default(),
            .transitions = Transitions.default(),
        };
    }

    /// Dark theme variant
    pub fn dark() Theme {
        return .{
            .colors = ColorPalette.dark(),
            .spacing = SpacingScale.default(),
            .typography = Typography.default(),
            .borders = Borders.default(),
            .shadows = Shadows.dark(),
            .transitions = Transitions.default(),
        };
    }

    /// Generate CSS variables
    pub fn toCssVariables(self: Theme, allocator: std.mem.Allocator) ![]const u8 {
        var list = std.ArrayList(u8).init(allocator);
        var writer = list.writer();

        try writer.writeAll(":root {\n");

        // Colors
        try writer.print("  --color-primary: {s};\n", .{self.colors.primary.toCss()});
        try writer.print("  --color-secondary: {s};\n", .{self.colors.secondary.toCss()});
        try writer.print("  --color-success: {s};\n", .{self.colors.success.toCss()});
        try writer.print("  --color-danger: {s};\n", .{self.colors.danger.toCss()});
        try writer.print("  --color-warning: {s};\n", .{self.colors.warning.toCss()});
        try writer.print("  --color-info: {s};\n", .{self.colors.info.toCss()});

        // Grays
        try writer.print("  --color-gray-50: {s};\n", .{self.colors.gray[0].toCss()});
        try writer.print("  --color-gray-100: {s};\n", .{self.colors.gray[1].toCss()});
        try writer.print("  --color-gray-200: {s};\n", .{self.colors.gray[2].toCss()});
        try writer.print("  --color-gray-300: {s};\n", .{self.colors.gray[3].toCss()});
        try writer.print("  --color-gray-400: {s};\n", .{self.colors.gray[4].toCss()});
        try writer.print("  --color-gray-500: {s};\n", .{self.colors.gray[5].toCss()});
        try writer.print("  --color-gray-600: {s};\n", .{self.colors.gray[6].toCss()});
        try writer.print("  --color-gray-700: {s};\n", .{self.colors.gray[7].toCss()});
        try writer.print("  --color-gray-800: {s};\n", .{self.colors.gray[8].toCss()});
        try writer.print("  --color-gray-900: {s};\n", .{self.colors.gray[9].toCss()});

        // Spacing
        for (self.spacing.scale, 0..) |space, i| {
            try writer.print("  --spacing-{d}: {s};\n", .{ i, space.toCss() });
        }

        // Typography
        try writer.print("  --font-sans: {s};\n", .{self.typography.font_family_sans});
        try writer.print("  --font-mono: {s};\n", .{self.typography.font_family_mono});

        for (self.typography.font_sizes, 0..) |size, i| {
            try writer.print("  --font-size-{d}: {s};\n", .{ i, size.toCss() });
        }

        // Borders
        try writer.print("  --border-radius-sm: {s};\n", .{self.borders.radius_sm.toCss()});
        try writer.print("  --border-radius-md: {s};\n", .{self.borders.radius_md.toCss()});
        try writer.print("  --border-radius-lg: {s};\n", .{self.borders.radius_lg.toCss()});

        try writer.writeAll("}\n");

        return list.toOwnedSlice();
    }
};
```

---

## COLOR PALETTE

```zig
pub const ColorPalette = struct {
    // Semantic colors
    primary: types.Color,
    secondary: types.Color,
    success: types.Color,
    danger: types.Color,
    warning: types.Color,
    info: types.Color,

    // Background and text
    background: types.Color,
    surface: types.Color,
    text: types.Color,
    text_secondary: types.Color,

    // Gray scale (10 shades)
    gray: [10]types.Color,

    pub fn light() ColorPalette {
        return .{
            .primary = .{ .hex = 0x3b82f6 },      // Blue
            .secondary = .{ .hex = 0x8b5cf6 },    // Purple
            .success = .{ .hex = 0x10b981 },      // Green
            .danger = .{ .hex = 0xef4444 },       // Red
            .warning = .{ .hex = 0xf59e0b },      // Orange
            .info = .{ .hex = 0x06b6d4 },         // Cyan

            .background = .{ .hex = 0xffffff },
            .surface = .{ .hex = 0xf9fafb },
            .text = .{ .hex = 0x111827 },
            .text_secondary = .{ .hex = 0x6b7280 },

            .gray = [10]types.Color{
                .{ .hex = 0xf9fafb },  // 50
                .{ .hex = 0xf3f4f6 },  // 100
                .{ .hex = 0xe5e7eb },  // 200
                .{ .hex = 0xd1d5db },  // 300
                .{ .hex = 0x9ca3af },  // 400
                .{ .hex = 0x6b7280 },  // 500
                .{ .hex = 0x4b5563 },  // 600
                .{ .hex = 0x374151 },  // 700
                .{ .hex = 0x1f2937 },  // 800
                .{ .hex = 0x111827 },  // 900
            },
        };
    }

    pub fn dark() ColorPalette {
        return .{
            .primary = .{ .hex = 0x60a5fa },
            .secondary = .{ .hex = 0xa78bfa },
            .success = .{ .hex = 0x34d399 },
            .danger = .{ .hex = 0xf87171 },
            .warning = .{ .hex = 0xfbbf24 },
            .info = .{ .hex = 0x22d3ee },

            .background = .{ .hex = 0x111827 },
            .surface = .{ .hex = 0x1f2937 },
            .text = .{ .hex = 0xf9fafb },
            .text_secondary = .{ .hex = 0x9ca3af },

            .gray = [10]types.Color{
                .{ .hex = 0x111827 },  // 50 (inverted)
                .{ .hex = 0x1f2937 },
                .{ .hex = 0x374151 },
                .{ .hex = 0x4b5563 },
                .{ .hex = 0x6b7280 },
                .{ .hex = 0x9ca3af },
                .{ .hex = 0xd1d5db },
                .{ .hex = 0xe5e7eb },
                .{ .hex = 0xf3f4f6 },
                .{ .hex = 0xf9fafb },  // 900 (inverted)
            },
        };
    }
};
```

---

## SPACING SCALE

```zig
pub const SpacingScale = struct {
    // Standard 8-point scale
    scale: [13]types.Length,

    pub fn default() SpacingScale {
        return .{
            .scale = [13]types.Length{
                .{ .rem = 0 },      // 0
                .{ .rem = 0.25 },   // 1 - 4px
                .{ .rem = 0.5 },    // 2 - 8px
                .{ .rem = 0.75 },   // 3 - 12px
                .{ .rem = 1 },      // 4 - 16px
                .{ .rem = 1.25 },   // 5 - 20px
                .{ .rem = 1.5 },    // 6 - 24px
                .{ .rem = 2 },      // 7 - 32px
                .{ .rem = 2.5 },    // 8 - 40px
                .{ .rem = 3 },      // 9 - 48px
                .{ .rem = 4 },      // 10 - 64px
                .{ .rem = 5 },      // 11 - 80px
                .{ .rem = 6 },      // 12 - 96px
            },
        };
    }

    pub fn get(self: SpacingScale, index: usize) types.Length {
        if (index >= self.scale.len) return self.scale[self.scale.len - 1];
        return self.scale[index];
    }
};
```

---

## TYPOGRAPHY

```zig
pub const Typography = struct {
    font_family_sans: []const u8,
    font_family_mono: []const u8,

    // Type scale
    font_sizes: [11]types.Length,
    line_heights: [11]types.Length,

    pub fn default() Typography {
        return .{
            .font_family_sans = "system-ui, -apple-system, sans-serif",
            .font_family_mono = "ui-monospace, monospace",

            .font_sizes = [11]types.Length{
                .{ .rem = 0.75 },   // xs
                .{ .rem = 0.875 },  // sm
                .{ .rem = 1 },      // base
                .{ .rem = 1.125 },  // lg
                .{ .rem = 1.25 },   // xl
                .{ .rem = 1.5 },    // 2xl
                .{ .rem = 1.875 },  // 3xl
                .{ .rem = 2.25 },   // 4xl
                .{ .rem = 3 },      // 5xl
                .{ .rem = 3.75 },   // 6xl
                .{ .rem = 4.5 },    // 7xl
            },

            .line_heights = [11]types.Length{
                .{ .rem = 1 },
                .{ .rem = 1.25 },
                .{ .rem = 1.5 },
                .{ .rem = 1.75 },
                .{ .rem = 1.75 },
                .{ .rem = 2 },
                .{ .rem = 2.25 },
                .{ .rem = 2.5 },
                .{ .rem = 1 },
                .{ .rem = 1 },
                .{ .rem = 1 },
            },
        };
    }
};
```

---

## BORDERS & SHADOWS

```zig
pub const Borders = struct {
    radius_sm: types.Length,
    radius_md: types.Length,
    radius_lg: types.Length,
    radius_xl: types.Length,
    radius_full: types.Length,

    width_thin: types.Length,
    width_normal: types.Length,
    width_thick: types.Length,

    pub fn default() Borders {
        return .{
            .radius_sm = .{ .px = 4 },
            .radius_md = .{ .px = 8 },
            .radius_lg = .{ .px = 12 },
            .radius_xl = .{ .px = 16 },
            .radius_full = .{ .px = 9999 },

            .width_thin = .{ .px = 1 },
            .width_normal = .{ .px = 2 },
            .width_thick = .{ .px = 4 },
        };
    }
};

pub const Shadows = struct {
    sm: []const u8,
    md: []const u8,
    lg: []const u8,
    xl: []const u8,

    pub fn default() Shadows {
        return .{
            .sm = "0 1px 2px 0 rgba(0, 0, 0, 0.05)",
            .md = "0 4px 6px -1px rgba(0, 0, 0, 0.1)",
            .lg = "0 10px 15px -3px rgba(0, 0, 0, 0.1)",
            .xl = "0 20px 25px -5px rgba(0, 0, 0, 0.1)",
        };
    }

    pub fn dark() Shadows {
        return .{
            .sm = "0 1px 2px 0 rgba(0, 0, 0, 0.3)",
            .md = "0 4px 6px -1px rgba(0, 0, 0, 0.5)",
            .lg = "0 10px 15px -3px rgba(0, 0, 0, 0.5)",
            .xl = "0 20px 25px -5px rgba(0, 0, 0, 0.5)",
        };
    }
};
```

---

## TRANSITIONS

```zig
pub const Transitions = struct {
    duration_fast: u32,    // ms
    duration_normal: u32,
    duration_slow: u32,

    easing_in: []const u8,
    easing_out: []const u8,
    easing_in_out: []const u8,

    pub fn default() Transitions {
        return .{
            .duration_fast = 150,
            .duration_normal = 300,
            .duration_slow = 500,

            .easing_in = "cubic-bezier(0.4, 0, 1, 1)",
            .easing_out = "cubic-bezier(0, 0, 0.2, 1)",
            .easing_in_out = "cubic-bezier(0.4, 0, 0.2, 1)",
        };
    }
};
```

---

## THEME PROVIDER

### Context Integration

```zig
// In AppContext
pub const AppContext = struct {
    // ... other fields
    theme: *const Theme,

    /// Get theme color
    pub fn color(self: *const AppContext, comptime semantic: []const u8) types.Color {
        return @field(self.theme.colors, semantic);
    }

    /// Get spacing value
    pub fn spacing(self: *const AppContext, index: usize) types.Length {
        return self.theme.spacing.get(index);
    }

    /// Get font size
    pub fn fontSize(self: *const AppContext, index: usize) types.Length {
        if (index >= self.theme.typography.font_sizes.len) {
            return self.theme.typography.font_sizes[self.theme.typography.font_sizes.len - 1];
        }
        return self.theme.typography.font_sizes[index];
    }
};
```

---

## DARK MODE SUPPORT

### Theme Switching

```zig
pub const ThemeManager = struct {
    allocator: std.mem.Allocator,
    current_theme: Theme,
    mode: ThemeMode,

    pub const ThemeMode = enum {
        light,
        dark,
        auto, // System preference
    };

    pub fn init(allocator: std.mem.Allocator) ThemeManager {
        return .{
            .allocator = allocator,
            .current_theme = Theme.default(),
            .mode = .light,
        };
    }

    pub fn setMode(self: *ThemeManager, mode: ThemeMode) void {
        self.mode = mode;
        self.current_theme = switch (mode) {
            .light => Theme.default(),
            .dark => Theme.dark(),
            .auto => self.getSystemTheme(),
        };
    }

    fn getSystemTheme(self: *ThemeManager) Theme {
        // In real implementation, would check system preference
        // via JS interop: window.matchMedia('(prefers-color-scheme: dark)')
        return Theme.default();
    }

    pub fn toggle(self: *ThemeManager) void {
        const new_mode: ThemeMode = if (self.mode == .light) .dark else .light;
        self.setMode(new_mode);
    }
};
```

### CSS Generation

```zig
pub fn generateThemeCss(theme: Theme, allocator: std.mem.Allocator) ![]const u8 {
    var list = std.ArrayList(u8).init(allocator);
    var writer = list.writer();

    // Generate CSS variables
    const vars = try theme.toCssVariables(allocator);
    defer allocator.free(vars);
    try writer.writeAll(vars);

    // Generate utility classes using theme
    try writer.writeAll("\n/* Theme-aware utilities */\n");
    try writer.writeAll(".bg-primary { background-color: var(--color-primary); }\n");
    try writer.writeAll(".text-primary { color: var(--color-primary); }\n");
    try writer.writeAll(".border-primary { border-color: var(--color-primary); }\n");

    // ... more utilities

    return list.toOwnedSlice();
}
```

---

## USAGE EXAMPLES

### Using the Theme

```zig
pub fn view(model: *const Model, ctx: *AppContext) Element(Msg) {
    return h.div(ctx, .{
        .style = styles.create(.{
            .background_color = ctx.color("primary"),
            .padding = ctx.spacing(4),
            .border_radius = ctx.theme.borders.radius_md,
        }),
    }, &.{
        h.h1(ctx, .{
            .style = styles.create(.{
                .font_size = ctx.fontSize(6), // 2xl
                .color = ctx.theme.colors.text,
            }),
        }, &.{
            h.text(ctx, "Themed Component"),
        }),
    });
}
```

### Theme Toggle

```zig
pub const Msg = enum {
    toggle_theme,
};

pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .toggle_theme => {
            model.theme_manager.toggle();
            return Effect.none;
        },
    };
}
```

---

## CONCLUSION

The theme system provides consistent design tokens, dark mode support, and an ergonomic API for coherent styles throughout the application.

**Links:**
- [← Previous: CSS Extraction](08-css-extraction.md)
- [→ Next: Routing](10-routing.md)
- [↑ Back to index](../README.md)
