# 25 - TRANSITIONS AND ANIMATIONS

**Document:** Transitions, Animations, and View Transitions
**Version:** 6.0.0
**Status:** Final
**Dependencies:** 03-virtual-dom.md, 05-css-types.md, 14-js-runtime.md

---

## OVERVIEW

ZUI provides a comprehensive system for animations and transitions:

1. **CSS Transitions:** Simple property animations via CSS
2. **Element Transitions:** Enter/leave animations for elements
3. **List Transitions:** Animated list reordering (FLIP)
4. **View Transitions:** Page-level transitions using the View Transitions API

---

## ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────┐
│                  TRANSITION SYSTEM                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 CSS Transitions                      │    │
│  │  .element { transition: opacity 300ms ease; }       │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Element Transitions                     │    │
│  │  <Transition>                                        │    │
│  │    - Enter: fade-in, slide-in, scale-up             │    │
│  │    - Leave: fade-out, slide-out, scale-down         │    │
│  │  </Transition>                                       │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               List Transitions                       │    │
│  │  <TransitionGroup>                                   │    │
│  │    - FLIP animations for reordering                 │    │
│  │    - Staggered enter/leave                          │    │
│  │  </TransitionGroup>                                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               View Transitions                       │    │
│  │  - Page-level morphing                              │    │
│  │  - Cross-document transitions                       │    │
│  │  - Uses View Transitions API                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## CSS TRANSITION UTILITIES

### Transition Styles

```zig
// src/css/transitions.zig
const std = @import("std");
const css = @import("types.zig");

/// Predefined transition presets
pub const transitions = struct {
    /// No transition
    pub const none = css.StyleRule{
        .transition = .none,
    };

    /// All properties
    pub const all = css.StyleRule{
        .transition_property = .all,
        .transition_duration = .{ .ms = 150 },
        .transition_timing_function = .ease,
    };

    /// Colors only (background, border, text)
    pub const colors = css.StyleRule{
        .transition_property = .{ .custom = "color, background-color, border-color, fill, stroke" },
        .transition_duration = .{ .ms = 150 },
        .transition_timing_function = .ease,
    };

    /// Opacity
    pub const opacity = css.StyleRule{
        .transition_property = .opacity,
        .transition_duration = .{ .ms = 150 },
        .transition_timing_function = .ease,
    };

    /// Box shadow
    pub const shadow = css.StyleRule{
        .transition_property = .{ .custom = "box-shadow" },
        .transition_duration = .{ .ms = 150 },
        .transition_timing_function = .ease,
    };

    /// Transform
    pub const transform = css.StyleRule{
        .transition_property = .transform,
        .transition_duration = .{ .ms = 150 },
        .transition_timing_function = .ease,
    };
};

/// Transition durations
pub const duration = struct {
    pub const fast = css.Duration{ .ms = 75 };
    pub const normal = css.Duration{ .ms = 150 };
    pub const slow = css.Duration{ .ms = 300 };
    pub const slower = css.Duration{ .ms = 500 };
    pub const slowest = css.Duration{ .ms = 700 };
};

/// Timing functions
pub const easing = struct {
    pub const linear = css.TimingFunction.linear;
    pub const ease = css.TimingFunction.ease;
    pub const ease_in = css.TimingFunction.ease_in;
    pub const ease_out = css.TimingFunction.ease_out;
    pub const ease_in_out = css.TimingFunction.ease_in_out;

    /// Custom cubic bezier curves
    pub const ease_in_back = css.TimingFunction{ .cubic_bezier = .{ 0.6, -0.28, 0.735, 0.045 } };
    pub const ease_out_back = css.TimingFunction{ .cubic_bezier = .{ 0.175, 0.885, 0.32, 1.275 } };
    pub const ease_in_out_back = css.TimingFunction{ .cubic_bezier = .{ 0.68, -0.55, 0.265, 1.55 } };

    pub const ease_in_circ = css.TimingFunction{ .cubic_bezier = .{ 0.6, 0.04, 0.98, 0.335 } };
    pub const ease_out_circ = css.TimingFunction{ .cubic_bezier = .{ 0.075, 0.82, 0.165, 1 } };

    pub const ease_in_expo = css.TimingFunction{ .cubic_bezier = .{ 0.95, 0.05, 0.795, 0.035 } };
    pub const ease_out_expo = css.TimingFunction{ .cubic_bezier = .{ 0.19, 1, 0.22, 1 } };
};
```

---

## ELEMENT TRANSITIONS

### Transition Component

```zig
// src/transitions/transition.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Configuration for element transitions
pub fn TransitionConfig(comptime Msg: type) type {
    return struct {
        /// Unique name for the transition (used for CSS class generation)
        name: []const u8 = "zui",

        /// Duration in milliseconds
        duration: u32 = 300,

        /// Timing function
        easing: TimingFunction = .ease_out,

        /// Transition mode
        mode: TransitionMode = .default,

        /// Whether element is shown
        show: bool,

        /// The element to transition
        children: []const vdom.Element(Msg),

        /// Callbacks
        on_before_enter: ?Msg = null,
        on_after_enter: ?Msg = null,
        on_before_leave: ?Msg = null,
        on_after_leave: ?Msg = null,

        /// Custom enter/leave classes (optional)
        enter_from_class: ?[]const u8 = null,
        enter_active_class: ?[]const u8 = null,
        enter_to_class: ?[]const u8 = null,
        leave_from_class: ?[]const u8 = null,
        leave_active_class: ?[]const u8 = null,
        leave_to_class: ?[]const u8 = null,
    };
}

pub const TransitionMode = enum {
    /// Enter and leave happen simultaneously
    default,
    /// Current element leaves first, then new enters
    out_in,
    /// New element enters first, then current leaves
    in_out,
};

pub const TimingFunction = enum {
    linear,
    ease,
    ease_in,
    ease_out,
    ease_in_out,
    ease_in_back,
    ease_out_back,
    ease_in_out_back,
};

/// Transition wrapper element
pub fn Transition(
    comptime Msg: type,
    ctx: *AppContext,
    config: TransitionConfig(Msg),
) vdom.Element(Msg) {
    return .{
        .transition = .{
            .name = config.name,
            .duration = config.duration,
            .easing = config.easing,
            .mode = config.mode,
            .show = config.show,
            .children = config.children,
            .callbacks = .{
                .on_before_enter = config.on_before_enter,
                .on_after_enter = config.on_after_enter,
                .on_before_leave = config.on_before_leave,
                .on_after_leave = config.on_after_leave,
            },
            .classes = .{
                .enter_from = config.enter_from_class,
                .enter_active = config.enter_active_class,
                .enter_to = config.enter_to_class,
                .leave_from = config.leave_from_class,
                .leave_active = config.leave_active_class,
                .leave_to = config.leave_to_class,
            },
        },
    };
}
```

### VDOM Extension

```zig
// Addition to src/vdom/element.zig

pub fn Element(comptime Msg: type) type {
    return union(enum) {
        // ... existing variants ...

        /// Transition wrapper for enter/leave animations
        transition: TransitionElement(Msg),

        /// Transition group for list animations
        transition_group: TransitionGroupElement(Msg),
    };
}

pub fn TransitionElement(comptime Msg: type) type {
    return struct {
        name: []const u8,
        duration: u32,
        easing: TimingFunction,
        mode: TransitionMode,
        show: bool,
        children: []Element(Msg),
        callbacks: TransitionCallbacks(Msg),
        classes: TransitionClasses,
    };
}

pub fn TransitionCallbacks(comptime Msg: type) type {
    return struct {
        on_before_enter: ?Msg = null,
        on_after_enter: ?Msg = null,
        on_before_leave: ?Msg = null,
        on_after_leave: ?Msg = null,
    };
}

pub const TransitionClasses = struct {
    enter_from: ?[]const u8 = null,
    enter_active: ?[]const u8 = null,
    enter_to: ?[]const u8 = null,
    leave_from: ?[]const u8 = null,
    leave_active: ?[]const u8 = null,
    leave_to: ?[]const u8 = null,
};
```

### Preset Transitions

```zig
// src/transitions/presets.zig
const std = @import("std");
const Transition = @import("transition.zig").Transition;
const TransitionConfig = @import("transition.zig").TransitionConfig;

/// Fade transition
pub fn Fade(
    comptime Msg: type,
    ctx: *AppContext,
    show: bool,
    children: []const vdom.Element(Msg),
) vdom.Element(Msg) {
    return Transition(Msg, ctx, .{
        .name = "fade",
        .show = show,
        .children = children,
        .duration = 200,
        .easing = .ease_out,
    });
}

/// Slide transition (from direction)
pub fn Slide(
    comptime Msg: type,
    ctx: *AppContext,
    config: SlideConfig(Msg),
) vdom.Element(Msg) {
    const name = switch (config.direction) {
        .up => "slide-up",
        .down => "slide-down",
        .left => "slide-left",
        .right => "slide-right",
    };

    return Transition(Msg, ctx, .{
        .name = name,
        .show = config.show,
        .children = config.children,
        .duration = config.duration,
        .easing = .ease_out,
    });
}

pub fn SlideConfig(comptime Msg: type) type {
    return struct {
        show: bool,
        children: []const vdom.Element(Msg),
        direction: Direction = .up,
        duration: u32 = 300,
    };
}

pub const Direction = enum { up, down, left, right };

/// Scale transition
pub fn Scale(
    comptime Msg: type,
    ctx: *AppContext,
    show: bool,
    children: []const vdom.Element(Msg),
) vdom.Element(Msg) {
    return Transition(Msg, ctx, .{
        .name = "scale",
        .show = show,
        .children = children,
        .duration = 200,
        .easing = .ease_out_back,
    });
}

/// Collapse transition (height animation)
pub fn Collapse(
    comptime Msg: type,
    ctx: *AppContext,
    show: bool,
    children: []const vdom.Element(Msg),
) vdom.Element(Msg) {
    return Transition(Msg, ctx, .{
        .name = "collapse",
        .show = show,
        .children = children,
        .duration = 300,
        .easing = .ease_out,
    });
}
```

---

## LIST TRANSITIONS (FLIP)

### TransitionGroup Component

```zig
// src/transitions/transition_group.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Configuration for list transitions
pub fn TransitionGroupConfig(comptime Msg: type) type {
    return struct {
        /// Name for transition CSS classes
        name: []const u8 = "list",

        /// Tag for the wrapper element
        tag: []const u8 = "div",

        /// Duration for move animations (FLIP)
        move_duration: u32 = 300,

        /// Duration for enter/leave
        enter_leave_duration: u32 = 200,

        /// Stagger delay between items
        stagger: u32 = 0,

        /// The list items (must have keys)
        children: []const vdom.Element(Msg),

        /// Whether to animate initial render
        appear: bool = false,
    };
}

/// Animated list wrapper
pub fn TransitionGroup(
    comptime Msg: type,
    ctx: *AppContext,
    config: TransitionGroupConfig(Msg),
) vdom.Element(Msg) {
    return .{
        .transition_group = .{
            .name = config.name,
            .tag = config.tag,
            .move_duration = config.move_duration,
            .enter_leave_duration = config.enter_leave_duration,
            .stagger = config.stagger,
            .children = config.children,
            .appear = config.appear,
        },
    };
}

pub fn TransitionGroupElement(comptime Msg: type) type {
    return struct {
        name: []const u8,
        tag: []const u8,
        move_duration: u32,
        enter_leave_duration: u32,
        stagger: u32,
        children: []Element(Msg),
        appear: bool,
    };
}
```

### FLIP Algorithm Implementation

```zig
// src/transitions/flip.zig
const std = @import("std");

/// FLIP (First, Last, Invert, Play) animation data
pub const FlipData = struct {
    /// Element key
    key: []const u8,

    /// First position (before change)
    first: Rect,

    /// Last position (after change)
    last: Rect,

    /// Calculated transform
    transform: Transform,

    pub const Rect = struct {
        x: f32,
        y: f32,
        width: f32,
        height: f32,
    };

    pub const Transform = struct {
        translateX: f32,
        translateY: f32,
        scaleX: f32,
        scaleY: f32,
    };

    /// Calculate the inversion transform
    pub fn calculateInvert(self: *FlipData) void {
        self.transform = .{
            .translateX = self.first.x - self.last.x,
            .translateY = self.first.y - self.last.y,
            .scaleX = if (self.last.width > 0) self.first.width / self.last.width else 1,
            .scaleY = if (self.last.height > 0) self.first.height / self.last.height else 1,
        };
    }
};

/// Manages FLIP animations for a transition group
pub const FlipManager = struct {
    elements: std.StringHashMap(FlipData),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) FlipManager {
        return .{
            .elements = std.StringHashMap(FlipData).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *FlipManager) void {
        self.elements.deinit();
    }

    /// Record first positions
    pub fn recordFirst(self: *FlipManager, keys: []const []const u8) void {
        for (keys) |key| {
            if (self.elements.getPtr(key)) |data| {
                data.first = getElementRect(key);
            } else {
                self.elements.put(key, .{
                    .key = key,
                    .first = getElementRect(key),
                    .last = .{ .x = 0, .y = 0, .width = 0, .height = 0 },
                    .transform = .{ .translateX = 0, .translateY = 0, .scaleX = 1, .scaleY = 1 },
                }) catch {};
            }
        }
    }

    /// Record last positions and calculate transforms
    pub fn recordLast(self: *FlipManager) void {
        var iter = self.elements.iterator();
        while (iter.next()) |entry| {
            entry.value_ptr.last = getElementRect(entry.key_ptr.*);
            entry.value_ptr.calculateInvert();
        }
    }

    /// Apply transforms and trigger animations
    pub fn play(self: *FlipManager, duration: u32) void {
        var iter = self.elements.iterator();
        while (iter.next()) |entry| {
            const data = entry.value_ptr.*;

            // Skip if no movement
            if (data.transform.translateX == 0 and data.transform.translateY == 0) {
                continue;
            }

            // Apply inverted transform immediately
            applyTransform(entry.key_ptr.*, data.transform);

            // Trigger animation to identity
            requestAnimationFrame(struct {
                fn callback() void {
                    animateToIdentity(entry.key_ptr.*, duration);
                }
            }.callback);
        }
    }

    fn getElementRect(key: []const u8) FlipData.Rect {
        // This will call into JS to get getBoundingClientRect
        return js_get_element_rect(key.ptr, key.len);
    }

    fn applyTransform(key: []const u8, transform: FlipData.Transform) void {
        js_apply_transform(
            key.ptr,
            key.len,
            transform.translateX,
            transform.translateY,
            transform.scaleX,
            transform.scaleY,
        );
    }

    fn animateToIdentity(key: []const u8, duration: u32) void {
        js_animate_to_identity(key.ptr, key.len, duration);
    }
};

// JS imports
extern "zui" fn js_get_element_rect(key_ptr: [*]const u8, key_len: usize) FlipData.Rect;
extern "zui" fn js_apply_transform(key_ptr: [*]const u8, key_len: usize, tx: f32, ty: f32, sx: f32, sy: f32) void;
extern "zui" fn js_animate_to_identity(key_ptr: [*]const u8, key_len: usize, duration: u32) void;
```

---

## VIEW TRANSITIONS API

### View Transition Support

```zig
// src/transitions/view_transition.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Check if View Transitions API is supported
pub fn isSupported() bool {
    return js_view_transitions_supported() != 0;
}

/// Start a view transition
pub fn startViewTransition(
    comptime Msg: type,
    callback: *const fn () void,
    on_ready: ?Msg,
    on_finished: ?Msg,
) Effect(Msg) {
    return Effect(Msg).viewTransition(.{
        .callback = callback,
        .on_ready = on_ready,
        .on_finished = on_finished,
    });
}

/// Assign view-transition-name to an element
pub fn viewTransitionName(name: []const u8) css.StyleRule {
    return css.StyleRule{
        .view_transition_name = name,
    };
}

/// Configure view transition for navigation
pub fn NavigationTransition(comptime Msg: type) type {
    return struct {
        /// Type of transition
        transition_type: TransitionType = .fade,

        /// Duration in milliseconds
        duration: u32 = 300,

        /// Callback when transition is ready
        on_ready: ?Msg = null,

        /// Callback when transition is finished
        on_finished: ?Msg = null,

        pub const TransitionType = enum {
            fade,
            slide_left,
            slide_right,
            slide_up,
            slide_down,
            morph,
            none,
        };
    };
}

extern "zui" fn js_view_transitions_supported() u8;
```

### View Transition Effect

```zig
// Addition to src/effects/effect.zig

pub fn Effect(comptime Msg: type) type {
    return union(enum) {
        // ... existing variants ...

        /// View transition effect
        view_transition: struct {
            callback: *const fn () void,
            on_ready: ?Msg,
            on_finished: ?Msg,
        },

        /// Navigate with view transition
        navigate_with_transition: struct {
            path: []const u8,
            transition_type: ViewTransitionType,
            on_complete: ?Msg,
        },
    };
}

pub const ViewTransitionType = enum {
    fade,
    slide_left,
    slide_right,
    slide_up,
    slide_down,
    morph,
    none,
};
```

---

## CSS GENERATION

### Transition CSS Classes

```css
/* Generated CSS for transitions */

/* Fade transition */
.fade-enter-from,
.fade-leave-to {
    opacity: 0;
}

.fade-enter-active,
.fade-leave-active {
    transition: opacity var(--zui-transition-duration, 200ms) var(--zui-transition-easing, ease-out);
}

.fade-enter-to,
.fade-leave-from {
    opacity: 1;
}

/* Slide transitions */
.slide-up-enter-from {
    opacity: 0;
    transform: translateY(20px);
}

.slide-up-leave-to {
    opacity: 0;
    transform: translateY(-20px);
}

.slide-up-enter-active,
.slide-up-leave-active {
    transition: opacity var(--zui-transition-duration, 300ms) var(--zui-transition-easing, ease-out),
                transform var(--zui-transition-duration, 300ms) var(--zui-transition-easing, ease-out);
}

/* Scale transition */
.scale-enter-from,
.scale-leave-to {
    opacity: 0;
    transform: scale(0.9);
}

.scale-enter-active,
.scale-leave-active {
    transition: opacity var(--zui-transition-duration, 200ms) var(--zui-transition-easing, ease-out),
                transform var(--zui-transition-duration, 200ms) cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

/* Collapse transition */
.collapse-enter-from,
.collapse-leave-to {
    height: 0;
    opacity: 0;
    overflow: hidden;
}

.collapse-enter-active,
.collapse-leave-active {
    transition: height var(--zui-transition-duration, 300ms) var(--zui-transition-easing, ease-out),
                opacity var(--zui-transition-duration, 300ms) var(--zui-transition-easing, ease-out);
    overflow: hidden;
}

/* List move transition */
.list-move {
    transition: transform var(--zui-move-duration, 300ms) var(--zui-transition-easing, ease-out);
}

.list-enter-from,
.list-leave-to {
    opacity: 0;
    transform: scale(0.8);
}

.list-enter-active,
.list-leave-active {
    transition: opacity var(--zui-transition-duration, 200ms) var(--zui-transition-easing, ease-out),
                transform var(--zui-transition-duration, 200ms) var(--zui-transition-easing, ease-out);
}

/* View Transitions */
::view-transition-old(root) {
    animation: var(--zui-view-transition-out, fade-out) var(--zui-view-transition-duration, 300ms) ease-out;
}

::view-transition-new(root) {
    animation: var(--zui-view-transition-in, fade-in) var(--zui-view-transition-duration, 300ms) ease-out;
}

@keyframes fade-out {
    from { opacity: 1; }
    to { opacity: 0; }
}

@keyframes fade-in {
    from { opacity: 0; }
    to { opacity: 1; }
}

@keyframes slide-out-left {
    from { transform: translateX(0); }
    to { transform: translateX(-100%); }
}

@keyframes slide-in-right {
    from { transform: translateX(100%); }
    to { transform: translateX(0); }
}
```

---

## JS RUNTIME INTEGRATION

```javascript
// www/zui.js - Transition handling

const ZuiTransitions = {
    // Active transitions
    activeTransitions: new Map(),

    // Transition element enter
    enter(element, name, duration, easing, callbacks) {
        const id = element.dataset.zuiId || this.generateId();
        element.dataset.zuiId = id;

        // Add enter-from class
        element.classList.add(`${name}-enter-from`);
        element.classList.add(`${name}-enter-active`);

        // Trigger callback
        if (callbacks.onBeforeEnter) {
            ZuiRuntime.dispatch(callbacks.onBeforeEnter);
        }

        // Force reflow
        element.offsetHeight;

        // Remove enter-from, add enter-to
        element.classList.remove(`${name}-enter-from`);
        element.classList.add(`${name}-enter-to`);

        // Set up transition end handler
        const onEnd = () => {
            element.classList.remove(`${name}-enter-active`);
            element.classList.remove(`${name}-enter-to`);
            element.removeEventListener('transitionend', onEnd);

            if (callbacks.onAfterEnter) {
                ZuiRuntime.dispatch(callbacks.onAfterEnter);
            }
        };

        element.addEventListener('transitionend', onEnd);
        this.activeTransitions.set(id, { element, type: 'enter', cleanup: onEnd });
    },

    // Transition element leave
    leave(element, name, duration, easing, callbacks) {
        const id = element.dataset.zuiId;

        // Add leave-from class
        element.classList.add(`${name}-leave-from`);
        element.classList.add(`${name}-leave-active`);

        // Trigger callback
        if (callbacks.onBeforeLeave) {
            ZuiRuntime.dispatch(callbacks.onBeforeLeave);
        }

        // Force reflow
        element.offsetHeight;

        // Remove leave-from, add leave-to
        element.classList.remove(`${name}-leave-from`);
        element.classList.add(`${name}-leave-to`);

        // Set up transition end handler
        const onEnd = () => {
            element.classList.remove(`${name}-leave-active`);
            element.classList.remove(`${name}-leave-to`);
            element.removeEventListener('transitionend', onEnd);

            // Remove element from DOM
            element.remove();

            if (callbacks.onAfterLeave) {
                ZuiRuntime.dispatch(callbacks.onAfterLeave);
            }
        };

        element.addEventListener('transitionend', onEnd);
        this.activeTransitions.set(id, { element, type: 'leave', cleanup: onEnd });
    },

    // FLIP animation for list reordering
    flipAnimate(elements, duration) {
        // Record first positions
        const firstRects = new Map();
        for (const el of elements) {
            const key = el.dataset.key;
            firstRects.set(key, el.getBoundingClientRect());
        }

        // Let DOM update happen
        requestAnimationFrame(() => {
            for (const el of elements) {
                const key = el.dataset.key;
                const first = firstRects.get(key);
                if (!first) continue;

                const last = el.getBoundingClientRect();

                // Calculate delta
                const deltaX = first.left - last.left;
                const deltaY = first.top - last.top;

                if (deltaX === 0 && deltaY === 0) continue;

                // Apply inverted transform
                el.style.transform = `translate(${deltaX}px, ${deltaY}px)`;
                el.style.transition = 'none';

                // Force reflow
                el.offsetHeight;

                // Animate to final position
                el.style.transform = '';
                el.style.transition = `transform ${duration}ms ease-out`;
            }
        });
    },

    // View Transition API wrapper
    async viewTransition(callback, onReady, onFinished) {
        if (!document.startViewTransition) {
            // Fallback: just call callback
            callback();
            if (onFinished) ZuiRuntime.dispatch(onFinished);
            return;
        }

        const transition = document.startViewTransition(callback);

        if (onReady) {
            transition.ready.then(() => {
                ZuiRuntime.dispatch(onReady);
            });
        }

        if (onFinished) {
            transition.finished.then(() => {
                ZuiRuntime.dispatch(onFinished);
            });
        }
    },

    generateId() {
        return `t-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    },
};
```

---

## USAGE EXAMPLES

### Basic Fade Transition

```zig
const zui = @import("zui");
const h = zui.html;
const transitions = zui.transitions;

pub fn Modal(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return transitions.Fade(Msg, ctx, model.show_modal, &.{
        h.div(ctx, .{ .class = "modal" }, &.{
            h.p(ctx, .{}, &.{ h.text(ctx, "Modal content") }),
            h.button(ctx, .{ .onClick = .close_modal }, &.{
                h.text(ctx, "Close"),
            }),
        }),
    });
}
```

### Slide Transition with Direction

```zig
pub fn Sidebar(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return transitions.Slide(Msg, ctx, .{
        .show = model.sidebar_open,
        .direction = .left,
        .duration = 300,
        .children = &.{
            h.nav(ctx, .{ .class = "sidebar" }, &.{
                // Sidebar content
            }),
        },
    });
}
```

### Animated List

```zig
pub fn TodoList(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    var items = ctx.frame_arena.allocator().alloc(
        zui.Element(Msg),
        model.todos.len,
    ) catch return .none;

    for (model.todos, 0..) |todo, i| {
        items[i] = h.li(ctx, .{
            .key = todo.id,
            .class = "todo-item",
        }, &.{
            h.text(ctx, todo.text),
            h.button(ctx, .{
                .onClick = .{ .delete_todo = todo.id },
            }, &.{ h.text(ctx, "×") }),
        });
    }

    return transitions.TransitionGroup(Msg, ctx, .{
        .name = "todo-list",
        .tag = "ul",
        .stagger = 50, // 50ms delay between items
        .children = items,
    });
}
```

### View Transition for Navigation

```zig
pub fn update(model: *Model, msg: Msg, ctx: *zui.AppContext) zui.Effect(Msg) {
    return switch (msg) {
        .navigate_to => |path| blk: {
            // Check if View Transitions are supported
            if (zui.transitions.isSupported()) {
                break :blk zui.Effect(Msg).navigate_with_transition(.{
                    .path = path,
                    .transition_type = .slide_left,
                    .on_complete = .navigation_complete,
                });
            } else {
                // Fallback to regular navigation
                break :blk zui.Effect(Msg).navigate(.{
                    .path = path,
                    .on_complete = .navigation_complete,
                });
            }
        },
        else => .none,
    };
}
```

### Custom Transition with Callbacks

```zig
pub fn ExpandableSection(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return h.div(ctx, .{ .class = "expandable" }, &.{
        h.button(ctx, .{
            .onClick = .toggle_expanded,
            .attrs = &.{
                .{ "aria-expanded", if (model.expanded) "true" else "false" },
            },
        }, &.{
            h.text(ctx, "Toggle"),
        }),

        transitions.Transition(Msg, ctx, .{
            .name = "expand",
            .show = model.expanded,
            .duration = 300,
            .easing = .ease_out,
            .on_before_enter = .section_expanding,
            .on_after_enter = .section_expanded,
            .on_before_leave = .section_collapsing,
            .on_after_leave = .section_collapsed,
            .children = &.{
                h.div(ctx, .{ .class = "expandable-content" }, &.{
                    // Content
                }),
            },
        }),
    });
}
```

---

## ACCESSIBILITY CONSIDERATIONS

```zig
// Respect user preference for reduced motion
pub fn shouldAnimate(ctx: *AppContext) bool {
    return !ctx.prefers_reduced_motion;
}

// Usage
pub fn AnimatedComponent(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    if (shouldAnimate(ctx)) {
        return transitions.Fade(Msg, ctx, model.show, &.{content});
    } else {
        // No animation - immediate show/hide
        return if (model.show) content else .none;
    }
}
```

---

## CONCLUSION

The transition system provides a comprehensive set of tools for creating smooth, accessible animations in ZUI applications. From simple CSS transitions to complex FLIP list animations and modern View Transitions, developers have full control over motion design while maintaining good performance and accessibility.

**Links:**
- [← Previous: Head Management](24-head-management.md)
- [→ Next: Slots](26-slots.md)
- [↑ Back to index](../README.md)
