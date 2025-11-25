# 21 - TAILWIND CSS INTEGRATION

**Document:** Tailwind CSS Integration
**Version:** 1.0.0
**Status:** Draft
**Dependencies:** 05-css-types.md, 07-css-builder.md, 08-css-extraction.md

---

## OVERVIEW

ZUI provides optional Tailwind CSS integration through a type-safe API that generates Tailwind class names at compile-time. This allows developers familiar with Tailwind to use their existing knowledge while maintaining ZUI's type-safety guarantees.

### Goals

1. **Type-safe Tailwind** - Compile-time validation of utility classes
2. **Zero runtime overhead** - Class strings computed at compile-time
3. **IDE support** - Full autocomplete and documentation
4. **Composable with ZSS** - Use alongside native style system
5. **No Node.js dependency** - Pure Zig implementation

### Non-Goals

1. Full Tailwind plugin ecosystem support
2. Runtime JIT compilation
3. Arbitrary value syntax (`w-[123px]`) - use ZSS for custom values

---

## ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────┐
│                    Component Code                            │
│  h.div(ctx, .{ .tw = .{ .flex, .p_4, .bg_blue_500 } }, ...) │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ (compile-time)
┌─────────────────────────────────────────────────────────────┐
│                   Tailwind Type System                       │
│  - Validates utility names                                   │
│  - Generates class string: "flex p-4 bg-blue-500"           │
│  - Tracks used classes for CSS extraction                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ (build-time)
┌─────────────────────────────────────────────────────────────┐
│                    CSS Generation                            │
│  Option A: Generate Tailwind-compatible CSS                  │
│  Option B: Output class list for Tailwind CLI               │
└─────────────────────────────────────────────────────────────┘
```

---

## TYPE-SAFE API

### Basic Usage

```zig
const zui = @import("zui");
const h = zui.html;
const tw = zui.tailwind;

pub fn view(model: *const Model, ctx: *zui.AppContext) zui.Element(Msg) {
    return h.div(ctx, .{
        .tw = tw.class(.{
            .flex,
            .flex_col,
            .items_center,
            .gap_4,
            .p_6,
            .bg_white,
            .rounded_lg,
            .shadow_md,
        }),
    }, &.{
        h.h1(ctx, .{
            .tw = tw.class(.{ .text_2xl, .font_bold, .text_gray_900 }),
        }, &.{
            h.text(ctx, "Hello Tailwind"),
        }),
    });
}
```

### Generated Output

The `tw.class()` function computes at compile-time:
```zig
// Input
tw.class(.{ .flex, .items_center, .gap_4 })

// Output (compile-time computed)
"flex items-center gap-4"
```

---

## UTILITY DEFINITIONS

### Layout

```zig
pub const Layout = enum {
    // Display
    block,
    inline_block,
    @"inline",
    flex,
    inline_flex,
    grid,
    inline_grid,
    hidden,

    // Flex Direction
    flex_row,
    flex_row_reverse,
    flex_col,
    flex_col_reverse,

    // Flex Wrap
    flex_wrap,
    flex_wrap_reverse,
    flex_nowrap,

    // Justify Content
    justify_start,
    justify_end,
    justify_center,
    justify_between,
    justify_around,
    justify_evenly,

    // Align Items
    items_start,
    items_end,
    items_center,
    items_baseline,
    items_stretch,

    // Align Self
    self_auto,
    self_start,
    self_end,
    self_center,
    self_stretch,

    // Position
    static,
    fixed,
    absolute,
    relative,
    sticky,
};
```

### Spacing

```zig
pub const Spacing = enum {
    // Padding (all sides)
    p_0, p_px, p_0_5, p_1, p_1_5, p_2, p_2_5, p_3, p_3_5, p_4,
    p_5, p_6, p_7, p_8, p_9, p_10, p_11, p_12, p_14, p_16,
    p_20, p_24, p_28, p_32, p_36, p_40, p_44, p_48, p_52,
    p_56, p_60, p_64, p_72, p_80, p_96,

    // Padding X (horizontal)
    px_0, px_px, px_0_5, px_1, px_1_5, px_2, px_2_5, px_3, px_3_5, px_4,
    px_5, px_6, px_7, px_8, px_9, px_10, px_11, px_12, px_14, px_16,
    px_20, px_24, px_28, px_32, px_36, px_40, px_44, px_48, px_52,
    px_56, px_60, px_64, px_72, px_80, px_96,

    // Padding Y (vertical)
    py_0, py_px, py_0_5, py_1, py_1_5, py_2, py_2_5, py_3, py_3_5, py_4,
    // ... same scale

    // Padding individual sides
    pt_0, pt_1, pt_2, // ... top
    pr_0, pr_1, pr_2, // ... right
    pb_0, pb_1, pb_2, // ... bottom
    pl_0, pl_1, pl_2, // ... left

    // Margin (same pattern as padding)
    m_0, m_px, m_0_5, m_1, m_auto,
    // mx_, my_, mt_, mr_, mb_, ml_ ...

    // Negative margins
    _m_1, _m_2, _m_3, _m_4, // -m-1, -m-2, etc.
    _mx_1, _mx_2, // ...

    // Gap
    gap_0, gap_px, gap_0_5, gap_1, gap_1_5, gap_2, gap_2_5, gap_3,
    gap_3_5, gap_4, gap_5, gap_6, gap_7, gap_8, gap_9, gap_10,
    gap_11, gap_12, gap_14, gap_16, gap_20, gap_24, gap_28, gap_32,

    gap_x_0, gap_x_1, gap_x_2, // ... horizontal gap
    gap_y_0, gap_y_1, gap_y_2, // ... vertical gap

    // Space between
    space_x_0, space_x_1, space_x_2, space_x_4, space_x_reverse,
    space_y_0, space_y_1, space_y_2, space_y_4, space_y_reverse,
};
```

### Sizing

```zig
pub const Sizing = enum {
    // Width
    w_0, w_px, w_0_5, w_1, w_1_5, w_2, w_2_5, w_3, w_3_5, w_4,
    w_5, w_6, w_7, w_8, w_9, w_10, w_11, w_12, w_14, w_16,
    w_20, w_24, w_28, w_32, w_36, w_40, w_44, w_48, w_52,
    w_56, w_60, w_64, w_72, w_80, w_96,
    w_auto, w_full, w_screen, w_min, w_max, w_fit,
    w_1_2, w_1_3, w_2_3, w_1_4, w_2_4, w_3_4,
    w_1_5, w_2_5, w_3_5, w_4_5,
    w_1_6, w_2_6, w_3_6, w_4_6, w_5_6,
    w_1_12, w_2_12, w_3_12, w_4_12, w_5_12, w_6_12,
    w_7_12, w_8_12, w_9_12, w_10_12, w_11_12,

    // Height (same scale)
    h_0, h_px, h_0_5, h_1, h_auto, h_full, h_screen, h_min, h_max, h_fit,
    // ...

    // Min/Max width
    min_w_0, min_w_full, min_w_min, min_w_max, min_w_fit,
    max_w_0, max_w_none, max_w_xs, max_w_sm, max_w_md, max_w_lg,
    max_w_xl, max_w_2xl, max_w_3xl, max_w_4xl, max_w_5xl, max_w_6xl,
    max_w_7xl, max_w_full, max_w_min, max_w_max, max_w_fit, max_w_prose,
    max_w_screen_sm, max_w_screen_md, max_w_screen_lg, max_w_screen_xl,
    max_w_screen_2xl,

    // Min/Max height
    min_h_0, min_h_full, min_h_screen, min_h_min, min_h_max, min_h_fit,
    max_h_0, max_h_full, max_h_screen, max_h_min, max_h_max, max_h_fit,
    // ... numeric scale
};
```

### Typography

```zig
pub const Typography = enum {
    // Font Size
    text_xs,      // 12px
    text_sm,      // 14px
    text_base,    // 16px
    text_lg,      // 18px
    text_xl,      // 20px
    text_2xl,     // 24px
    text_3xl,     // 30px
    text_4xl,     // 36px
    text_5xl,     // 48px
    text_6xl,     // 60px
    text_7xl,     // 72px
    text_8xl,     // 96px
    text_9xl,     // 128px

    // Font Weight
    font_thin,        // 100
    font_extralight,  // 200
    font_light,       // 300
    font_normal,      // 400
    font_medium,      // 500
    font_semibold,    // 600
    font_bold,        // 700
    font_extrabold,   // 800
    font_black,       // 900

    // Font Style
    italic,
    not_italic,

    // Text Alignment
    text_left,
    text_center,
    text_right,
    text_justify,
    text_start,
    text_end,

    // Text Decoration
    underline,
    overline,
    line_through,
    no_underline,

    // Text Transform
    uppercase,
    lowercase,
    capitalize,
    normal_case,

    // Line Height
    leading_none,      // 1
    leading_tight,     // 1.25
    leading_snug,      // 1.375
    leading_normal,    // 1.5
    leading_relaxed,   // 1.625
    leading_loose,     // 2
    leading_3, leading_4, leading_5, leading_6, leading_7,
    leading_8, leading_9, leading_10,

    // Letter Spacing
    tracking_tighter,
    tracking_tight,
    tracking_normal,
    tracking_wide,
    tracking_wider,
    tracking_widest,

    // Whitespace
    whitespace_normal,
    whitespace_nowrap,
    whitespace_pre,
    whitespace_pre_line,
    whitespace_pre_wrap,
    whitespace_break_spaces,

    // Word Break
    break_normal,
    break_words,
    break_all,
    break_keep,

    // Truncate
    truncate,
    text_ellipsis,
    text_clip,
};
```

### Colors

Colors use a structured approach for type safety:

```zig
pub const Colors = struct {
    // Text colors
    pub const text = ColorScale("text");
    // Background colors
    pub const bg = ColorScale("bg");
    // Border colors
    pub const border = ColorScale("border");
    // Ring colors (focus rings)
    pub const ring = ColorScale("ring");
    // Divide colors
    pub const divide = ColorScale("divide");
};

fn ColorScale(comptime prefix: []const u8) type {
    return struct {
        // Special colors
        pub const inherit = prefix ++ "-inherit";
        pub const current = prefix ++ "-current";
        pub const transparent = prefix ++ "-transparent";
        pub const black = prefix ++ "-black";
        pub const white = prefix ++ "-white";

        // Slate
        pub const slate_50 = prefix ++ "-slate-50";
        pub const slate_100 = prefix ++ "-slate-100";
        pub const slate_200 = prefix ++ "-slate-200";
        pub const slate_300 = prefix ++ "-slate-300";
        pub const slate_400 = prefix ++ "-slate-400";
        pub const slate_500 = prefix ++ "-slate-500";
        pub const slate_600 = prefix ++ "-slate-600";
        pub const slate_700 = prefix ++ "-slate-700";
        pub const slate_800 = prefix ++ "-slate-800";
        pub const slate_900 = prefix ++ "-slate-900";
        pub const slate_950 = prefix ++ "-slate-950";

        // Gray
        pub const gray_50 = prefix ++ "-gray-50";
        pub const gray_100 = prefix ++ "-gray-100";
        // ... same pattern for all shades

        // Zinc, Neutral, Stone, Red, Orange, Amber, Yellow,
        // Lime, Green, Emerald, Teal, Cyan, Sky, Blue,
        // Indigo, Violet, Purple, Fuchsia, Pink, Rose
        // ... all following same pattern
    };
}
```

### Usage with Colors

```zig
const tw = zui.tailwind;

h.div(ctx, .{
    .tw = tw.class(.{
        .bg_blue_500,        // Using enum directly
        tw.text.gray_900,    // Using ColorScale
        tw.bg.white,
        tw.border.gray_200,
    }),
}, &.{...});
```

### Borders

```zig
pub const Borders = enum {
    // Border Width
    border,           // 1px
    border_0,
    border_2,
    border_4,
    border_8,
    border_t, border_t_0, border_t_2, border_t_4, border_t_8,
    border_r, border_r_0, border_r_2, border_r_4, border_r_8,
    border_b, border_b_0, border_b_2, border_b_4, border_b_8,
    border_l, border_l_0, border_l_2, border_l_4, border_l_8,
    border_x, border_x_0, border_x_2, border_x_4, border_x_8,
    border_y, border_y_0, border_y_2, border_y_4, border_y_8,

    // Border Style
    border_solid,
    border_dashed,
    border_dotted,
    border_double,
    border_hidden,
    border_none,

    // Border Radius
    rounded_none,
    rounded_sm,
    rounded,
    rounded_md,
    rounded_lg,
    rounded_xl,
    rounded_2xl,
    rounded_3xl,
    rounded_full,

    // Individual corners
    rounded_t_none, rounded_t_sm, rounded_t, rounded_t_md, rounded_t_lg,
    rounded_r_none, rounded_r_sm, rounded_r, rounded_r_md, rounded_r_lg,
    rounded_b_none, rounded_b_sm, rounded_b, rounded_b_md, rounded_b_lg,
    rounded_l_none, rounded_l_sm, rounded_l, rounded_l_md, rounded_l_lg,
    rounded_tl_none, rounded_tl_sm, rounded_tl, rounded_tl_md, rounded_tl_lg,
    rounded_tr_none, rounded_tr_sm, rounded_tr, rounded_tr_md, rounded_tr_lg,
    rounded_bl_none, rounded_bl_sm, rounded_bl, rounded_bl_md, rounded_bl_lg,
    rounded_br_none, rounded_br_sm, rounded_br, rounded_br_md, rounded_br_lg,

    // Divide Width
    divide_x, divide_x_0, divide_x_2, divide_x_4, divide_x_8,
    divide_y, divide_y_0, divide_y_2, divide_y_4, divide_y_8,
    divide_x_reverse,
    divide_y_reverse,

    // Divide Style
    divide_solid,
    divide_dashed,
    divide_dotted,
    divide_double,
    divide_none,

    // Ring
    ring, ring_0, ring_1, ring_2, ring_4, ring_8, ring_inset,
};
```

### Effects

```zig
pub const Effects = enum {
    // Shadow
    shadow_sm,
    shadow,
    shadow_md,
    shadow_lg,
    shadow_xl,
    shadow_2xl,
    shadow_inner,
    shadow_none,

    // Opacity
    opacity_0,
    opacity_5,
    opacity_10,
    opacity_20,
    opacity_25,
    opacity_30,
    opacity_40,
    opacity_50,
    opacity_60,
    opacity_70,
    opacity_75,
    opacity_80,
    opacity_90,
    opacity_95,
    opacity_100,

    // Mix Blend Mode
    mix_blend_normal,
    mix_blend_multiply,
    mix_blend_screen,
    mix_blend_overlay,
    mix_blend_darken,
    mix_blend_lighten,
    mix_blend_color_dodge,
    mix_blend_color_burn,
    mix_blend_hard_light,
    mix_blend_soft_light,
    mix_blend_difference,
    mix_blend_exclusion,
    mix_blend_hue,
    mix_blend_saturation,
    mix_blend_color,
    mix_blend_luminosity,
};
```

### Transitions and Animations

```zig
pub const Transitions = enum {
    // Transition Property
    transition_none,
    transition_all,
    transition,
    transition_colors,
    transition_opacity,
    transition_shadow,
    transition_transform,

    // Duration
    duration_0,
    duration_75,
    duration_100,
    duration_150,
    duration_200,
    duration_300,
    duration_500,
    duration_700,
    duration_1000,

    // Timing Function
    ease_linear,
    ease_in,
    ease_out,
    ease_in_out,

    // Delay
    delay_0,
    delay_75,
    delay_100,
    delay_150,
    delay_200,
    delay_300,
    delay_500,
    delay_700,
    delay_1000,

    // Animation
    animate_none,
    animate_spin,
    animate_ping,
    animate_pulse,
    animate_bounce,
};
```

### Transforms

```zig
pub const Transforms = enum {
    // Scale
    scale_0, scale_50, scale_75, scale_90, scale_95, scale_100,
    scale_105, scale_110, scale_125, scale_150,
    scale_x_0, scale_x_50, scale_x_75, scale_x_90, scale_x_95, scale_x_100,
    scale_x_105, scale_x_110, scale_x_125, scale_x_150,
    scale_y_0, scale_y_50, scale_y_75, scale_y_90, scale_y_95, scale_y_100,
    scale_y_105, scale_y_110, scale_y_125, scale_y_150,

    // Rotate
    rotate_0, rotate_1, rotate_2, rotate_3, rotate_6, rotate_12,
    rotate_45, rotate_90, rotate_180,
    _rotate_1, _rotate_2, _rotate_3, _rotate_6, _rotate_12,
    _rotate_45, _rotate_90, _rotate_180,

    // Translate
    translate_x_0, translate_x_px, translate_x_0_5, translate_x_1,
    translate_x_1_2, translate_x_1_3, translate_x_2_3, translate_x_1_4,
    translate_x_full,
    _translate_x_1, _translate_x_2, _translate_x_full,
    translate_y_0, translate_y_px, translate_y_0_5, translate_y_1,
    translate_y_1_2, translate_y_full,
    _translate_y_1, _translate_y_2, _translate_y_full,

    // Skew
    skew_x_0, skew_x_1, skew_x_2, skew_x_3, skew_x_6, skew_x_12,
    _skew_x_1, _skew_x_2, _skew_x_3, _skew_x_6, _skew_x_12,
    skew_y_0, skew_y_1, skew_y_2, skew_y_3, skew_y_6, skew_y_12,
    _skew_y_1, _skew_y_2, _skew_y_3, _skew_y_6, _skew_y_12,

    // Transform Origin
    origin_center,
    origin_top,
    origin_top_right,
    origin_right,
    origin_bottom_right,
    origin_bottom,
    origin_bottom_left,
    origin_left,
    origin_top_left,
};
```

### Interactivity

```zig
pub const Interactivity = enum {
    // Cursor
    cursor_auto,
    cursor_default,
    cursor_pointer,
    cursor_wait,
    cursor_text,
    cursor_move,
    cursor_help,
    cursor_not_allowed,
    cursor_none,
    cursor_context_menu,
    cursor_progress,
    cursor_cell,
    cursor_crosshair,
    cursor_vertical_text,
    cursor_alias,
    cursor_copy,
    cursor_no_drop,
    cursor_grab,
    cursor_grabbing,
    cursor_all_scroll,
    cursor_col_resize,
    cursor_row_resize,
    cursor_n_resize,
    cursor_e_resize,
    cursor_s_resize,
    cursor_w_resize,
    cursor_ne_resize,
    cursor_nw_resize,
    cursor_se_resize,
    cursor_sw_resize,
    cursor_ew_resize,
    cursor_ns_resize,
    cursor_nesw_resize,
    cursor_nwse_resize,
    cursor_zoom_in,
    cursor_zoom_out,

    // User Select
    select_none,
    select_text,
    select_all,
    select_auto,

    // Pointer Events
    pointer_events_none,
    pointer_events_auto,

    // Resize
    resize_none,
    resize,
    resize_x,
    resize_y,

    // Scroll Behavior
    scroll_auto,
    scroll_smooth,

    // Touch Action
    touch_auto,
    touch_none,
    touch_pan_x,
    touch_pan_left,
    touch_pan_right,
    touch_pan_y,
    touch_pan_up,
    touch_pan_down,
    touch_pinch_zoom,
    touch_manipulation,
};
```

---

## RESPONSIVE VARIANTS

Tailwind's responsive prefixes are supported via the variant system:

```zig
const tw = zui.tailwind;

h.div(ctx, .{
    .tw = tw.class(.{
        // Base styles (mobile-first)
        .w_full,
        .flex_col,
        .p_4,

        // sm: 640px+
        tw.sm(.{ .w_1_2, .p_6 }),

        // md: 768px+
        tw.md(.{ .w_1_3, .flex_row, .p_8 }),

        // lg: 1024px+
        tw.lg(.{ .w_1_4 }),

        // xl: 1280px+
        tw.xl(.{ .max_w_6xl }),

        // 2xl: 1536px+
        tw.@"2xl"(.{ .max_w_7xl }),
    }),
}, &.{...});
```

### Generated Output

```
w-full flex-col p-4 sm:w-1/2 sm:p-6 md:w-1/3 md:flex-row md:p-8 lg:w-1/4 xl:max-w-6xl 2xl:max-w-7xl
```

---

## STATE VARIANTS

### Hover, Focus, Active

```zig
h.button(ctx, .{
    .tw = tw.class(.{
        .bg_blue_500,
        .text_white,
        .px_4,
        .py_2,
        .rounded,
        .transition_colors,

        tw.hover(.{ .bg_blue_600 }),
        tw.focus(.{ .ring_2, .ring_blue_400, .outline_none }),
        tw.active(.{ .bg_blue_700 }),
    }),
    .onClick = .submit,
}, &.{
    h.text(ctx, "Submit"),
});
```

### All State Variants

```zig
pub const Variant = struct {
    // Mouse states
    pub fn hover(classes: anytype) []const u8 { ... }
    pub fn focus(classes: anytype) []const u8 { ... }
    pub fn focus_within(classes: anytype) []const u8 { ... }
    pub fn focus_visible(classes: anytype) []const u8 { ... }
    pub fn active(classes: anytype) []const u8 { ... }
    pub fn visited(classes: anytype) []const u8 { ... }
    pub fn target(classes: anytype) []const u8 { ... }

    // Form states
    pub fn disabled(classes: anytype) []const u8 { ... }
    pub fn enabled(classes: anytype) []const u8 { ... }
    pub fn checked(classes: anytype) []const u8 { ... }
    pub fn indeterminate(classes: anytype) []const u8 { ... }
    pub fn default(classes: anytype) []const u8 { ... }
    pub fn required(classes: anytype) []const u8 { ... }
    pub fn valid(classes: anytype) []const u8 { ... }
    pub fn invalid(classes: anytype) []const u8 { ... }
    pub fn in_range(classes: anytype) []const u8 { ... }
    pub fn out_of_range(classes: anytype) []const u8 { ... }
    pub fn placeholder_shown(classes: anytype) []const u8 { ... }
    pub fn autofill(classes: anytype) []const u8 { ... }
    pub fn read_only(classes: anytype) []const u8 { ... }

    // Pseudo-elements
    pub fn before(classes: anytype) []const u8 { ... }
    pub fn after(classes: anytype) []const u8 { ... }
    pub fn placeholder(classes: anytype) []const u8 { ... }
    pub fn file(classes: anytype) []const u8 { ... }
    pub fn marker(classes: anytype) []const u8 { ... }
    pub fn selection(classes: anytype) []const u8 { ... }
    pub fn first_line(classes: anytype) []const u8 { ... }
    pub fn first_letter(classes: anytype) []const u8 { ... }
    pub fn backdrop(classes: anytype) []const u8 { ... }

    // Structural
    pub fn first(classes: anytype) []const u8 { ... }
    pub fn last(classes: anytype) []const u8 { ... }
    pub fn only(classes: anytype) []const u8 { ... }
    pub fn odd(classes: anytype) []const u8 { ... }
    pub fn even(classes: anytype) []const u8 { ... }
    pub fn first_of_type(classes: anytype) []const u8 { ... }
    pub fn last_of_type(classes: anytype) []const u8 { ... }
    pub fn only_of_type(classes: anytype) []const u8 { ... }
    pub fn empty(classes: anytype) []const u8 { ... }

    // Parent/sibling states
    pub fn group_hover(classes: anytype) []const u8 { ... }
    pub fn group_focus(classes: anytype) []const u8 { ... }
    pub fn peer_hover(classes: anytype) []const u8 { ... }
    pub fn peer_focus(classes: anytype) []const u8 { ... }
    pub fn peer_checked(classes: anytype) []const u8 { ... }

    // Dark mode
    pub fn dark(classes: anytype) []const u8 { ... }

    // Print
    pub fn print(classes: anytype) []const u8 { ... }

    // Motion preferences
    pub fn motion_safe(classes: anytype) []const u8 { ... }
    pub fn motion_reduce(classes: anytype) []const u8 { ... }

    // Contrast preferences
    pub fn contrast_more(classes: anytype) []const u8 { ... }
    pub fn contrast_less(classes: anytype) []const u8 { ... }

    // Orientation
    pub fn portrait(classes: anytype) []const u8 { ... }
    pub fn landscape(classes: anytype) []const u8 { ... }

    // LTR/RTL
    pub fn ltr(classes: anytype) []const u8 { ... }
    pub fn rtl(classes: anytype) []const u8 { ... }
};
```

### Combining Variants

```zig
h.input(ctx, .{
    .tw = tw.class(.{
        .border,
        .border_gray_300,
        .rounded,
        .px_3,
        .py_2,

        // Hover state
        tw.hover(.{ .border_gray_400 }),

        // Focus state
        tw.focus(.{ .border_blue_500, .ring_2, .ring_blue_200, .outline_none }),

        // Dark mode
        tw.dark(.{
            .bg_gray_800,
            .border_gray_600,
            .text_white,
        }),

        // Dark mode + focus
        tw.dark(tw.focus(.{ .border_blue_400, .ring_blue_800 })),

        // Invalid state
        tw.invalid(.{ .border_red_500 }),

        // Disabled state
        tw.disabled(.{ .bg_gray_100, .cursor_not_allowed, .opacity_50 }),
    }),
    .type = .text,
    .placeholder = "Enter text...",
});
```

---

## GROUP AND PEER MODIFIERS

### Group Hover

```zig
h.div(ctx, .{
    .tw = tw.class(.{ .group }),  // Mark container as group
}, &.{
    h.div(ctx, .{
        .tw = tw.class(.{
            .bg_white,
            .p_4,
            .rounded,
            .shadow,
            .transition_shadow,

            // React to parent hover
            tw.group_hover(.{ .shadow_lg }),
        }),
    }, &.{
        h.span(ctx, .{
            .tw = tw.class(.{
                .text_gray_600,
                tw.group_hover(.{ .text_blue_600 }),
            }),
        }, &.{
            h.text(ctx, "Hover the card"),
        }),
    }),
});
```

### Peer States

```zig
h.div(ctx, .{
    .tw = tw.class(.{ .flex, .flex_col, .gap_1 }),
}, &.{
    h.input(ctx, .{
        .tw = tw.class(.{ .peer, .border, .rounded, .px_3, .py_2 }),
        .type = .email,
        .placeholder = "email@example.com",
    }),
    h.span(ctx, .{
        .tw = tw.class(.{
            .text_red_500,
            .text_sm,
            .hidden,
            // Show when peer is invalid
            tw.peer_invalid(.{ .block }),
        }),
    }, &.{
        h.text(ctx, "Please enter a valid email"),
    }),
});
```

---

## COMPOSABILITY WITH ZSS

Tailwind utilities can be combined with native ZSS styles:

```zig
const styles = zui.css.styles;
const tw = zui.tailwind;

pub fn view(model: *const Model, ctx: *zui.AppContext) zui.Element(Msg) {
    return h.div(ctx, .{
        // Tailwind for common utilities
        .tw = tw.class(.{
            .flex,
            .items_center,
            .gap_4,
            .p_6,
        }),
        // ZSS for complex/custom styles
        .style = styles.create(.{
            .grid_template_columns = .{ .repeat = .{ .auto_fit, .{ .minmax = .{ .{ .rem = 15 }, .fr(1) } } } },
            .background = .{ .linear_gradient = .{
                .angle = .{ .deg = 135 },
                .stops = &.{
                    .{ .color = .{ .hex = 0x667eea } },
                    .{ .color = .{ .hex = 0x764ba2 } },
                },
            }},
        }),
    }, &.{...});
}
```

### Resolution Order

1. `.style` (ZSS) generates atomic classes and inline specificity
2. `.tw` generates Tailwind utility classes
3. `.class` adds raw class strings (escape hatch)

When conflicts occur:
- ZSS takes precedence (higher specificity via atomic classes)
- Later Tailwind classes override earlier ones (standard CSS cascade)

---

## CSS GENERATION

ZUI uses **Tailwind CLI** for CSS generation. This leverages the official, well-maintained Tailwind toolchain rather than reimplementing CSS generation.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         COMPILE-TIME (Zig)                              │
│  tw.class(.{ .flex, .p_4, tw.hover(.{ .bg_blue_600 }) })               │
│                              │                                          │
│                              ▼                                          │
│            Generates string: "flex p-4 hover:bg-blue-600"              │
│            (type validation at compile-time)                            │
└─────────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         BUILD-TIME (zig build)                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    TailwindExtractor                               │ │
│  │  - Scans .zig files for tw.class(), tw.hover(), tw.sm(), etc.     │ │
│  │  - Extracts class names from source code                          │ │
│  │  - Outputs .build/tailwind-classes.html                           │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                               │                                         │
│                               ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                    Tailwind CLI (npx tailwindcss)                  │ │
│  │  - Reads extracted classes                                        │ │
│  │  - Generates optimized CSS with only used utilities               │ │
│  │  - Outputs www/tailwind.css                                       │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### TailwindExtractor (Simplified)

The `TailwindExtractor` scans Zig source files to extract class names, then outputs a content file that Tailwind CLI can read to generate optimized CSS.

**Key insight:** We don't need to implement CSS generation ourselves—Tailwind CLI already does this perfectly. Our job is simply to extract which classes are used.

```zig
// build/tailwind_extractor.zig

const std = @import("std");

pub const TailwindExtractor = struct {
    allocator: std.mem.Allocator,
    classes: std.StringHashMap(void),

    pub fn init(allocator: std.mem.Allocator) TailwindExtractor {
        return .{
            .allocator = allocator,
            .classes = std.StringHashMap(void).init(allocator),
        };
    }

    pub fn deinit(self: *TailwindExtractor) void {
        self.classes.deinit();
    }

    /// Scan a directory for .zig files and extract Tailwind classes
    pub fn scanDirectory(self: *TailwindExtractor, dir_path: []const u8) !void {
        var dir = try std.fs.cwd().openDir(dir_path, .{ .iterate = true });
        defer dir.close();

        var walker = try dir.walk(self.allocator);
        defer walker.deinit();

        while (try walker.next()) |entry| {
            if (entry.kind == .file and std.mem.endsWith(u8, entry.basename, ".zig")) {
                const full_path = try std.fs.path.join(self.allocator, &.{ dir_path, entry.path });
                defer self.allocator.free(full_path);
                try self.extractFromFile(full_path);
            }
        }
    }

    /// Extract classes from a single file
    pub fn extractFromFile(self: *TailwindExtractor, file_path: []const u8) !void {
        const source = try std.fs.cwd().readFileAlloc(self.allocator, file_path, 10 * 1024 * 1024);
        defer self.allocator.free(source);

        var i: usize = 0;
        while (i < source.len) {
            // Find tw.class(, tw.hover(, tw.sm(, etc.
            if (self.findTwCall(source[i..])) |offset| {
                const content = self.extractBraceContent(source[i + offset..]) catch {
                    i += 1;
                    continue;
                };
                self.parseEnums(content) catch {};
                i += offset + content.len;
            } else {
                i += 1;
            }
        }
    }

    fn findTwCall(self: *TailwindExtractor, source: []const u8) ?usize {
        _ = self;
        const patterns = [_][]const u8{
            "tw.class(",  "tw.hover(",  "tw.focus(",  "tw.active(",
            "tw.disabled(", "tw.dark(",  "tw.sm(",     "tw.md(",
            "tw.lg(",     "tw.xl(",     "tw.@\"2xl\"(",
            "tw.group_hover(", "tw.peer_hover(", "tw.peer_invalid(",
        };
        for (patterns) |p| {
            if (std.mem.startsWith(u8, source, p)) return p.len;
        }
        return null;
    }

    fn extractBraceContent(self: *TailwindExtractor, source: []const u8) ![]const u8 {
        _ = self;
        // Find .{
        var i: usize = 0;
        while (i < source.len) : (i += 1) {
            if (source[i] == '.' and i + 1 < source.len and source[i + 1] == '{') break;
        }
        if (i >= source.len) return error.NotFound;

        const start = i + 2;
        var depth: u32 = 1;
        i = start;
        while (i < source.len and depth > 0) : (i += 1) {
            if (source[i] == '{') depth += 1;
            if (source[i] == '}') depth -= 1;
        }
        return source[start .. i - 1];
    }

    fn parseEnums(self: *TailwindExtractor, content: []const u8) !void {
        var iter = std.mem.tokenizeAny(u8, content, ", \t\n\r");
        while (iter.next()) |token| {
            if (std.mem.startsWith(u8, token, "tw.")) continue; // Skip nested calls
            var name = token;
            if (name.len > 0 and name[0] == '.') name = name[1..];

            // Convert snake_case to kebab-case: p_4 → p-4
            const css_class = try self.enumToCss(name);
            try self.classes.put(css_class, {});
        }
    }

    fn enumToCss(self: *TailwindExtractor, name: []const u8) ![]const u8 {
        var result = std.ArrayList(u8).init(self.allocator);
        var start: usize = 0;

        // Handle negative: _m_4 → -m-4
        if (name.len > 0 and name[0] == '_') {
            try result.append('-');
            start = 1;
        }

        for (name[start..]) |c| {
            try result.append(if (c == '_') '-' else c);
        }
        return result.toOwnedSlice();
    }

    /// Write extracted classes to a file Tailwind can read
    pub fn writeContentFile(self: *TailwindExtractor, output_path: []const u8) !void {
        var file = try std.fs.cwd().createFile(output_path, .{});
        defer file.close();

        const writer = file.writer();

        // Write as HTML that Tailwind's content scanner can parse
        try writer.writeAll("<!-- ZUI Tailwind Classes (auto-generated) -->\n");
        try writer.writeAll("<div class=\"");

        var iter = self.classes.keyIterator();
        var first = true;
        while (iter.next()) |key| {
            if (!first) try writer.writeAll(" ");
            try writer.writeAll(key.*);
            first = false;
        }

        try writer.writeAll("\"></div>\n");
    }
};

/// Build step to extract Tailwind classes
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    if (args.len < 3) {
        std.debug.print("Usage: tailwind-extract <src-dir> <output-file>\n", .{});
        return;
    }

    var extractor = TailwindExtractor.init(allocator);
    defer extractor.deinit();

    try extractor.scanDirectory(args[1]);
    try extractor.writeContentFile(args[2]);

    std.debug.print("Extracted {d} Tailwind classes to {s}\n", .{ extractor.classes.count(), args[2] });
}
```

---

### Tailwind CLI Integration

Instead of reimplementing Tailwind's CSS generation (hundreds of utilities, variants, media queries), we use the official **Tailwind CLI** to generate CSS. This approach:

- Uses the battle-tested official implementation
- Supports all Tailwind features automatically
- Stays up-to-date with Tailwind releases
- Reduces our codebase complexity significantly

**Requirement:** Node.js for the build step (Tailwind CLI).

---

### Build System Integration

The build process has two steps:

1. **Extract** - ZUI scans Zig files and outputs a content file with used classes
2. **Generate** - Tailwind CLI reads the content file and generates optimized CSS

```zig
// build.zig

const std = @import("std");

pub fn build(b: *std.Build) void {
    // ... existing build configuration ...

    // ============== TAILWIND CSS GENERATION ==============

    const tailwind_step = b.step("tailwind", "Generate Tailwind CSS");

    // Step 1: Extract Tailwind classes from Zig source files
    const extract_exe = b.addExecutable(.{
        .name = "tailwind-extract",
        .root_source_file = b.path("build/tailwind_extractor.zig"),
        .target = b.graph.host,
    });

    const extract_run = b.addRunArtifact(extract_exe);
    extract_run.addArgs(&.{ "src", ".build/tailwind-content.html" });

    // Step 2: Run Tailwind CLI to generate CSS
    const tailwind_run = b.addSystemCommand(&.{
        "npx", "tailwindcss",
        "-i", "src/tailwind-input.css",
        "-o", "www/tailwind.css",
        "--minify",
    });
    tailwind_run.step.dependOn(&extract_run.step);

    tailwind_step.dependOn(&tailwind_run.step);

    // Main build depends on Tailwind
    // wasm_step.dependOn(tailwind_step);
}
```

---

### Project Configuration

#### tailwind.config.js

```js
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    // Classes extracted by ZUI at build time
    '.build/tailwind-content.html',
    // Fallback: scan Zig files directly (for raw class strings)
    './src/**/*.zig',
  ],
  darkMode: 'class',  // Use .dark class for dark mode
  theme: {
    extend: {
      // Custom theme extensions here
      colors: {
        primary: {
          50: '#eff6ff',
          100: '#dbeafe',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
        },
      },
    },
  },
  plugins: [
    // Add Tailwind plugins as needed
    // require('@tailwindcss/forms'),
    // require('@tailwindcss/typography'),
  ],
}
```

#### src/tailwind-input.css

```css
/* src/tailwind-input.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom components or utilities can be added here */
@layer components {
  /* Example: reusable button styles */
}
```

---

### Build Commands

```bash
# Development: watch mode with hot reload
npx tailwindcss -i src/tailwind-input.css -o www/tailwind.css --watch

# Production: minified output
zig build tailwind
# Or directly:
npx tailwindcss -i src/tailwind-input.css -o www/tailwind.css --minify

# Full build (ZUI + Tailwind)
zig build && npx tailwindcss -i src/tailwind-input.css -o www/tailwind.css --minify
```

---

### Output Structure

```
project/
├── .build/
│   └── tailwind-content.html    # Auto-generated: extracted classes
├── src/
│   ├── tailwind-input.css       # Tailwind directives
│   └── **/*.zig                 # Your Zig source files
├── www/
│   ├── tailwind.css             # Generated CSS (Tailwind)
│   └── styles.css               # ZSS styles (if using both)
├── tailwind.config.js           # Tailwind configuration
└── build.zig
```

---

### Example: Extracted Content File

When you run `zig build tailwind`, the extractor scans your Zig files and produces:

```html
<!-- .build/tailwind-content.html -->
<!-- ZUI Tailwind Classes (auto-generated) -->
<div class="flex flex-col items-center p-4 gap-4 bg-white rounded-lg shadow-md
            hover:shadow-lg text-gray-800 dark:bg-gray-800 dark:text-white
            sm:flex-row sm:p-6 md:p-8 lg:max-w-4xl"></div>
```

Tailwind CLI reads this file and generates only the CSS for these classes.

---

## CONFIGURATION

### Custom Theme in Zig

While Tailwind theme is configured in `tailwind.config.js`, you can define type-safe custom values in Zig that map to your theme:

```zig
// src/tailwind/custom.zig

/// Custom color palette matching tailwind.config.js
pub const CustomColors = enum {
    primary_50,
    primary_100,
    primary_500,
    primary_600,
    primary_700,
};

/// Extension for tw module
pub fn customClass(comptime classes: anytype) []const u8 {
    // Compile-time string generation for custom classes
    // Maps Zig enums to Tailwind class names
}
```

### Using Custom Colors

```zig
h.div(ctx, .{
    .tw = tw.class(.{
        .bg_primary_500,  // Uses custom primary color from theme
        .text_white,
    }),
}, &.{...});
```

---

## ESCAPE HATCHES

### Raw Class Strings

For edge cases or classes not yet in the type system:

```zig
h.div(ctx, .{
    .tw = tw.class(.{
        .flex,
        .items_center,
    }),
    // Escape hatch for raw classes
    .class = "custom-animation my-special-class",
}, &.{...});
```

### Arbitrary Values (via ZSS)

Tailwind's arbitrary value syntax (`w-[123px]`) is not supported. Use ZSS instead:

```zig
h.div(ctx, .{
    // Use ZSS for arbitrary values
    .style = styles.create(.{
        .width = .{ .px = 123 },
        .margin_top = .{ .px = 17 },
    }),
    // Use Tailwind for standard utilities
    .tw = tw.class(.{ .flex, .gap_4 }),
}, &.{...});
```

---

## COMPLETE EXAMPLE

```zig
const zui = @import("zui");
const h = zui.html;
const tw = zui.tailwind;
const styles = zui.css.styles;

pub fn Card(ctx: *zui.AppContext, props: CardProps) zui.Element(Msg) {
    return h.article(ctx, .{
        .tw = tw.class(.{
            // Layout
            .flex,
            .flex_col,
            .overflow_hidden,

            // Sizing
            .w_full,
            tw.md(.{ .w_80 }),

            // Appearance
            .bg_white,
            .rounded_xl,
            .shadow_md,

            // Transitions
            .transition_all,
            .duration_200,

            // Hover effects
            tw.hover(.{
                .shadow_xl,
                ._translate_y_1,
            }),

            // Dark mode
            tw.dark(.{
                .bg_gray_800,
            }),
        }),
    }, &.{
        // Image
        if (props.image) |img|
            h.img(ctx, .{
                .src = img.src,
                .alt = img.alt,
                .tw = tw.class(.{
                    .w_full,
                    .h_48,
                    .object_cover,
                }),
            })
        else
            h.none(),

        // Content
        h.div(ctx, .{
            .tw = tw.class(.{ .p_6 }),
        }, &.{
            h.h3(ctx, .{
                .tw = tw.class(.{
                    .text_lg,
                    .font_semibold,
                    .text_gray_900,
                    tw.dark(.{ .text_white }),
                }),
            }, &.{
                h.text(ctx, props.title),
            }),

            h.p(ctx, .{
                .tw = tw.class(.{
                    .mt_2,
                    .text_gray_600,
                    .text_sm,
                    .leading_relaxed,
                    tw.dark(.{ .text_gray_300 }),
                }),
            }, &.{
                h.text(ctx, props.description),
            }),
        }),

        // Actions
        h.div(ctx, .{
            .tw = tw.class(.{
                .px_6,
                .py_4,
                .border_t,
                .border_gray_100,
                tw.dark(.{ .border_gray_700 }),
                .flex,
                .justify_end,
                .gap_2,
            }),
        }, &.{
            h.button(ctx, .{
                .tw = tw.class(.{
                    .px_4,
                    .py_2,
                    .text_sm,
                    .font_medium,
                    .text_gray_600,
                    .rounded_lg,
                    .transition_colors,
                    tw.hover(.{ .bg_gray_100, .text_gray_900 }),
                    tw.dark(.{ .text_gray_300 }),
                    tw.dark(tw.hover(.{ .bg_gray_700, .text_white })),
                }),
                .onClick = .{ .dismiss = props.id },
            }, &.{
                h.text(ctx, "Dismiss"),
            }),
            h.button(ctx, .{
                .tw = tw.class(.{
                    .px_4,
                    .py_2,
                    .text_sm,
                    .font_medium,
                    .text_white,
                    .bg_blue_500,
                    .rounded_lg,
                    .transition_colors,
                    tw.hover(.{ .bg_blue_600 }),
                    tw.focus(.{ .ring_2, .ring_blue_400, .ring_offset_2 }),
                    tw.active(.{ .bg_blue_700 }),
                }),
                .onClick = .{ .select = props.id },
            }, &.{
                h.text(ctx, "Learn More"),
            }),
        }),
    });
}
```

---

## MIGRATION FROM TAILWIND PROJECTS

### Strategy

1. Copy existing Tailwind class strings
2. Convert to type-safe format using editor tooling (planned)
3. Gradually replace with native syntax

### Converter Tool (Future)

```bash
# Planned: Convert Tailwind strings to Zig
zig build convert-tailwind -- "flex items-center gap-4 p-6 bg-blue-500"
# Output: .{ .flex, .items_center, .gap_4, .p_6, .bg_blue_500 }
```

---

## COMPARISON

| Feature | Native ZSS | Tailwind Integration |
|---------|-----------|---------------------|
| Type safety | Full | Full |
| Arbitrary values | Yes | No (use ZSS) |
| CSS Variables | Yes | Limited |
| Complex selectors | Yes | No |
| Media queries | Via theme | Via variants |
| Animations | Full keyframes | Predefined |
| Bundle size | Optimal | Larger (more classes) |
| Familiarity | New API | Familiar for TW users |

### When to Use Each

**Use Native ZSS for:**
- Custom/arbitrary values
- Complex layouts (grid templates)
- Custom animations
- CSS variables
- Design system foundations

**Use Tailwind Integration for:**
- Rapid prototyping
- Teams familiar with Tailwind
- Standard spacing/typography
- State variants (hover, focus, etc.)
- Responsive design

**Best Practice: Combine Both**
- ZSS for design system foundations
- Tailwind for component variations and utilities

---

## LINKS

- [← Previous: Browser APIs](20-browser-apis.md)
- [→ Next: Testing](17-tests.md)
- [Back to Index](../README.md)
