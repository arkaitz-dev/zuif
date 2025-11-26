# 23 - PORTALS

**Document:** Portals for Out-of-Tree Rendering
**Version:** 6.0.0
**Status:** Final
**Dependencies:** 03-virtual-dom.md, 14-js-runtime.md

---

## OVERVIEW

Portals provide a way to render children into a DOM node that exists outside the DOM hierarchy of the parent component. This is essential for UI patterns like modals, tooltips, dropdowns, and notifications that need to "break out" of their container's styling constraints (overflow, z-index, positioning).

---

## ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────┐
│                      PORTAL SYSTEM                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  VDOM Tree:                    Actual DOM:                  │
│  ┌─────────────┐               ┌─────────────┐              │
│  │    App      │               │    App      │              │
│  │  ┌───────┐  │               │  ┌───────┐  │              │
│  │  │ Page  │  │               │  │ Page  │  │              │
│  │  │┌─────┐│  │               │  │       │  │              │
│  │  ││Portal││  │    ═══▶      │  │       │  │              │
│  │  ││Modal ││  │               │  └───────┘  │              │
│  │  │└─────┘│  │               └─────────────┘              │
│  │  └───────┘  │                      │                      │
│  └─────────────┘               ┌──────▼──────┐              │
│                                │ #modal-root │              │
│  Events bubble                 │  ┌───────┐  │              │
│  through VDOM ◀────────────── │  │ Modal │  │              │
│  tree, not DOM                 │  └───────┘  │              │
│                                └─────────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## PORTAL ELEMENT

### Core Definition

```zig
// src/portal/portal.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Portal target specification
pub const PortalTarget = union(enum) {
    /// Render into element with this ID
    id: []const u8,

    /// Render into document.body
    body,

    /// Render into a specific DOM node reference
    node: u32,

    /// Create a new container with this ID if not exists
    create: struct {
        id: []const u8,
        parent: PortalTarget,
    },
};

/// Portal configuration
pub fn PortalConfig(comptime Msg: type) type {
    return struct {
        /// Where to render the portal content
        target: PortalTarget = .body,

        /// Children to render in the portal
        children: []const vdom.Element(Msg),

        /// Optional: key for portal (useful for animations)
        key: ?[]const u8 = null,

        /// Whether portal should capture focus when mounted
        trap_focus: bool = false,

        /// Whether clicking outside should trigger a message
        on_click_outside: ?Msg = null,

        /// Whether pressing Escape should trigger a message
        on_escape: ?Msg = null,

        /// Z-index for the portal container
        z_index: ?i32 = null,
    };
}
```

### VDOM Extension

```zig
// Addition to src/vdom/element.zig

pub fn Element(comptime Msg: type) type {
    return union(enum) {
        // ... existing variants ...

        /// Portal - renders children outside the DOM tree
        portal: struct {
            target: PortalTarget,
            children: []Element(Msg),
            key: ?[]const u8,
            options: PortalOptions(Msg),
        },
    };
}

pub fn PortalOptions(comptime Msg: type) type {
    return struct {
        trap_focus: bool = false,
        on_click_outside: ?Msg = null,
        on_escape: ?Msg = null,
        z_index: ?i32 = null,
        /// Restore focus to this element when portal unmounts
        restore_focus: bool = true,
    };
}
```

### HTML DSL Integration

```zig
// src/html/portal.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");
const Element = vdom.Element;
const PortalTarget = @import("../portal/portal.zig").PortalTarget;

/// Create a portal
pub fn portal(
    comptime Msg: type,
    ctx: *AppContext,
    config: PortalConfig(Msg),
) Element(Msg) {
    return .{
        .portal = .{
            .target = config.target,
            .children = config.children,
            .key = config.key,
            .options = .{
                .trap_focus = config.trap_focus,
                .on_click_outside = config.on_click_outside,
                .on_escape = config.on_escape,
                .z_index = config.z_index,
            },
        },
    };
}

/// Convenience: portal to body
pub fn portalToBody(
    comptime Msg: type,
    ctx: *AppContext,
    children: []const Element(Msg),
) Element(Msg) {
    return portal(Msg, ctx, .{
        .target = .body,
        .children = children,
    });
}

/// Convenience: portal to specific element
pub fn portalTo(
    comptime Msg: type,
    ctx: *AppContext,
    target_id: []const u8,
    children: []const Element(Msg),
) Element(Msg) {
    return portal(Msg, ctx, .{
        .target = .{ .id = target_id },
        .children = children,
    });
}
```

---

## MODAL COMPONENT

### High-Level Modal API

```zig
// src/components/modal.zig
const std = @import("std");
const zui = @import("../zui.zig");
const h = zui.html;
const Element = zui.Element;

pub fn ModalConfig(comptime Msg: type) type {
    return struct {
        /// Whether modal is open
        is_open: bool,

        /// Message to send when modal should close
        on_close: Msg,

        /// Modal title (optional)
        title: ?[]const u8 = null,

        /// Modal size
        size: ModalSize = .medium,

        /// Whether clicking backdrop closes modal
        close_on_backdrop: bool = true,

        /// Whether pressing Escape closes modal
        close_on_escape: bool = true,

        /// Whether to show close button
        show_close_button: bool = true,

        /// Custom class for modal container
        class: ?[]const u8 = null,

        /// Content of the modal
        children: []const Element(Msg),

        /// Footer content (optional)
        footer: ?[]const Element(Msg) = null,
    };
}

pub const ModalSize = enum {
    small,   // 400px
    medium,  // 600px
    large,   // 800px
    full,    // 100% - 2rem
};

/// Modal component with portal, backdrop, and accessibility
pub fn Modal(
    comptime Msg: type,
    ctx: *zui.AppContext,
    config: ModalConfig(Msg),
) Element(Msg) {
    if (!config.is_open) {
        return .none;
    }

    const size_class = switch (config.size) {
        .small => "modal-sm",
        .medium => "modal-md",
        .large => "modal-lg",
        .full => "modal-full",
    };

    return h.portal(Msg, ctx, .{
        .target = .body,
        .trap_focus = true,
        .on_escape = if (config.close_on_escape) config.on_close else null,
        .on_click_outside = if (config.close_on_backdrop) config.on_close else null,
        .z_index = 1000,
        .children = &.{
            // Backdrop
            h.div(ctx, .{
                .class = "modal-backdrop",
                .style = modalBackdropStyle(),
                .attrs = &.{
                    .{ "aria-hidden", "true" },
                },
            }, &.{}),

            // Modal container
            h.div(ctx, .{
                .class = joinClasses("modal-container", size_class, config.class),
                .style = modalContainerStyle(config.size),
                .attrs = &.{
                    .{ "role", "dialog" },
                    .{ "aria-modal", "true" },
                    .{ "aria-labelledby", "modal-title" },
                },
            }, &.{
                // Header
                if (config.title != null or config.show_close_button)
                    h.div(ctx, .{ .class = "modal-header" }, &.{
                        if (config.title) |title|
                            h.h2(ctx, .{
                                .id = "modal-title",
                                .class = "modal-title",
                            }, &.{ h.text(ctx, title) })
                        else
                            .none,

                        if (config.show_close_button)
                            h.button(ctx, .{
                                .class = "modal-close",
                                .onClick = config.on_close,
                                .attrs = &.{
                                    .{ "aria-label", "Close modal" },
                                },
                            }, &.{
                                h.text(ctx, "×"),
                            })
                        else
                            .none,
                    })
                else
                    .none,

                // Body
                h.div(ctx, .{ .class = "modal-body" }, config.children),

                // Footer
                if (config.footer) |footer|
                    h.div(ctx, .{ .class = "modal-footer" }, footer)
                else
                    .none,
            }),
        },
    });
}

fn modalBackdropStyle() zui.css.StyleRule {
    return zui.css.styles.create(.{
        .position = .fixed,
        .inset = .{ .px = 0 },
        .background_color = .{ .rgba = .{ 0, 0, 0, 0.5 } },
        .z_index = -1,
    });
}

fn modalContainerStyle(size: ModalSize) zui.css.StyleRule {
    const max_width = switch (size) {
        .small => .{ .px = 400 },
        .medium => .{ .px = 600 },
        .large => .{ .px = 800 },
        .full => .{ .calc = "100% - 2rem" },
    };

    return zui.css.styles.create(.{
        .position = .fixed,
        .top = .{ .percent = 50 },
        .left = .{ .percent = 50 },
        .transform = .{ .translate = .{ .{ .percent = -50 }, .{ .percent = -50 } } },
        .max_width = max_width,
        .width = .{ .percent = 100 },
        .max_height = .{ .calc = "100vh - 4rem" },
        .overflow_y = .auto,
        .background_color = .{ .var = "color-surface" },
        .border_radius = .{ .px = 8 },
        .box_shadow = .{ .lg = true },
    });
}
```

---

## TOOLTIP COMPONENT

```zig
// src/components/tooltip.zig
const std = @import("std");
const zui = @import("../zui.zig");
const h = zui.html;
const Element = zui.Element;

pub const TooltipPosition = enum {
    top,
    bottom,
    left,
    right,
};

pub fn TooltipConfig(comptime Msg: type) type {
    return struct {
        /// Content to display in tooltip
        content: []const u8,

        /// Position relative to trigger
        position: TooltipPosition = .top,

        /// Delay before showing (ms)
        show_delay: u32 = 200,

        /// Delay before hiding (ms)
        hide_delay: u32 = 0,

        /// The trigger element
        trigger: Element(Msg),

        /// Whether tooltip is currently visible
        is_visible: bool = false,

        /// Messages for show/hide
        on_show: ?Msg = null,
        on_hide: ?Msg = null,
    };
}

pub fn Tooltip(
    comptime Msg: type,
    ctx: *zui.AppContext,
    config: TooltipConfig(Msg),
) Element(Msg) {
    return h.div(ctx, .{
        .class = "tooltip-wrapper",
        .style = zui.css.styles.create(.{
            .position = .relative,
            .display = .inline_block,
        }),
        .onMouseEnter = config.on_show,
        .onMouseLeave = config.on_hide,
    }, &.{
        // Trigger element
        config.trigger,

        // Tooltip portal (only when visible)
        if (config.is_visible)
            h.portal(Msg, ctx, .{
                .target = .body,
                .z_index = 9999,
                .children = &.{
                    h.div(ctx, .{
                        .class = "tooltip",
                        .style = tooltipStyle(config.position),
                        .attrs = &.{
                            .{ "role", "tooltip" },
                        },
                    }, &.{
                        h.text(ctx, config.content),
                    }),
                },
            })
        else
            .none,
    });
}

fn tooltipStyle(position: TooltipPosition) zui.css.StyleRule {
    return zui.css.styles.create(.{
        .position = .absolute,
        .padding = .{ .px = 8 },
        .background_color = .{ .hex = 0x333333 },
        .color = .{ .hex = 0xffffff },
        .border_radius = .{ .px = 4 },
        .font_size = .{ .rem = 0.875 },
        .white_space = .nowrap,
        .pointer_events = .none,
        // Position will be set by JS based on trigger position
    });
}
```

---

## DROPDOWN COMPONENT

```zig
// src/components/dropdown.zig
const std = @import("std");
const zui = @import("../zui.zig");
const h = zui.html;
const Element = zui.Element;

pub fn DropdownConfig(comptime Msg: type) type {
    return struct {
        /// Whether dropdown is open
        is_open: bool,

        /// Message to toggle open state
        on_toggle: Msg,

        /// Message when clicking outside
        on_close: Msg,

        /// The trigger element (button, etc.)
        trigger: Element(Msg),

        /// Dropdown menu items
        items: []const DropdownItem(Msg),

        /// Alignment of dropdown menu
        align: Alignment = .start,

        /// Position of dropdown menu
        position: Position = .bottom,
    };
}

pub fn DropdownItem(comptime Msg: type) type {
    return struct {
        label: []const u8,
        onClick: Msg,
        icon: ?[]const u8 = null,
        disabled: bool = false,
        separator_after: bool = false,
    };
}

pub const Alignment = enum { start, end, center };
pub const Position = enum { top, bottom };

pub fn Dropdown(
    comptime Msg: type,
    ctx: *zui.AppContext,
    config: DropdownConfig(Msg),
) Element(Msg) {
    return h.div(ctx, .{
        .class = "dropdown",
        .style = zui.css.styles.create(.{
            .position = .relative,
            .display = .inline_block,
        }),
    }, &.{
        // Trigger with click handler
        h.div(ctx, .{
            .onClick = config.on_toggle,
        }, &.{config.trigger}),

        // Dropdown menu (portal when open)
        if (config.is_open)
            h.portal(Msg, ctx, .{
                .target = .body,
                .on_click_outside = config.on_close,
                .on_escape = config.on_close,
                .z_index = 1000,
                .children = &.{
                    h.div(ctx, .{
                        .class = "dropdown-menu",
                        .style = dropdownMenuStyle(),
                        .attrs = &.{
                            .{ "role", "menu" },
                        },
                    }, renderItems(Msg, ctx, config.items)),
                },
            })
        else
            .none,
    });
}

fn renderItems(
    comptime Msg: type,
    ctx: *zui.AppContext,
    items: []const DropdownItem(Msg),
) []Element(Msg) {
    var result = ctx.frame_arena.allocator().alloc(Element(Msg), items.len * 2) catch return &.{};
    var i: usize = 0;

    for (items) |item| {
        result[i] = h.button(ctx, .{
            .class = if (item.disabled) "dropdown-item disabled" else "dropdown-item",
            .onClick = if (!item.disabled) item.onClick else null,
            .disabled = item.disabled,
            .attrs = &.{
                .{ "role", "menuitem" },
            },
        }, &.{
            if (item.icon) |icon|
                h.span(ctx, .{ .class = "dropdown-icon" }, &.{
                    h.text(ctx, icon),
                })
            else
                .none,
            h.text(ctx, item.label),
        });
        i += 1;

        if (item.separator_after) {
            result[i] = h.hr(ctx, .{ .class = "dropdown-separator" });
            i += 1;
        }
    }

    return result[0..i];
}

fn dropdownMenuStyle() zui.css.StyleRule {
    return zui.css.styles.create(.{
        .position = .absolute,
        .min_width = .{ .px = 160 },
        .padding = .{ .py = .{ .px = 4 } },
        .background_color = .{ .var = "color-surface" },
        .border = .{ .width = .{ .px = 1 }, .color = .{ .var = "color-border" } },
        .border_radius = .{ .px = 4 },
        .box_shadow = .{ .md = true },
    });
}
```

---

## NOTIFICATION/TOAST COMPONENT

```zig
// src/components/toast.zig
const std = @import("std");
const zui = @import("../zui.zig");
const h = zui.html;
const Element = zui.Element;

pub const ToastType = enum {
    info,
    success,
    warning,
    @"error",
};

pub const ToastPosition = enum {
    top_left,
    top_center,
    top_right,
    bottom_left,
    bottom_center,
    bottom_right,
};

pub fn Toast(comptime Msg: type) type {
    return struct {
        id: u32,
        message: []const u8,
        toast_type: ToastType = .info,
        duration_ms: u32 = 5000, // 0 = no auto-dismiss
        dismissible: bool = true,
    };
}

pub fn ToastContainerConfig(comptime Msg: type) type {
    return struct {
        toasts: []const Toast(Msg),
        position: ToastPosition = .top_right,
        on_dismiss: *const fn (u32) Msg, // toast id
    };
}

pub fn ToastContainer(
    comptime Msg: type,
    ctx: *zui.AppContext,
    config: ToastContainerConfig(Msg),
) Element(Msg) {
    if (config.toasts.len == 0) {
        return .none;
    }

    return h.portal(Msg, ctx, .{
        .target = .{ .create = .{ .id = "toast-container", .parent = .body } },
        .z_index = 9999,
        .children = &.{
            h.div(ctx, .{
                .class = "toast-container",
                .style = toastContainerStyle(config.position),
                .attrs = &.{
                    .{ "aria-live", "polite" },
                    .{ "aria-atomic", "true" },
                },
            }, renderToasts(Msg, ctx, config.toasts, config.on_dismiss)),
        },
    });
}

fn renderToasts(
    comptime Msg: type,
    ctx: *zui.AppContext,
    toasts: []const Toast(Msg),
    on_dismiss: *const fn (u32) Msg,
) []Element(Msg) {
    var result = ctx.frame_arena.allocator().alloc(Element(Msg), toasts.len) catch return &.{};

    for (toasts, 0..) |toast, i| {
        result[i] = h.div(ctx, .{
            .class = toastClass(toast.toast_type),
            .style = toastStyle(),
            .key = std.fmt.allocPrint(ctx.frame_arena.allocator(), "toast-{d}", .{toast.id}) catch "toast",
            .attrs = &.{
                .{ "role", "alert" },
            },
        }, &.{
            h.span(ctx, .{ .class = "toast-message" }, &.{
                h.text(ctx, toast.message),
            }),
            if (toast.dismissible)
                h.button(ctx, .{
                    .class = "toast-dismiss",
                    .onClick = on_dismiss(toast.id),
                    .attrs = &.{
                        .{ "aria-label", "Dismiss notification" },
                    },
                }, &.{
                    h.text(ctx, "×"),
                })
            else
                .none,
        });
    }

    return result;
}

fn toastClass(toast_type: ToastType) []const u8 {
    return switch (toast_type) {
        .info => "toast toast-info",
        .success => "toast toast-success",
        .warning => "toast toast-warning",
        .@"error" => "toast toast-error",
    };
}

fn toastContainerStyle(position: ToastPosition) zui.css.StyleRule {
    const base = zui.css.styles.create(.{
        .position = .fixed,
        .display = .flex,
        .flex_direction = .column,
        .gap = .{ .px = 8 },
        .padding = .{ .px = 16 },
        .max_width = .{ .px = 400 },
        .width = .{ .percent = 100 },
    });

    return switch (position) {
        .top_left => base.merge(.{ .top = .{ .px = 0 }, .left = .{ .px = 0 } }),
        .top_center => base.merge(.{ .top = .{ .px = 0 }, .left = .{ .percent = 50 }, .transform = .{ .translateX = .{ .percent = -50 } } }),
        .top_right => base.merge(.{ .top = .{ .px = 0 }, .right = .{ .px = 0 } }),
        .bottom_left => base.merge(.{ .bottom = .{ .px = 0 }, .left = .{ .px = 0 } }),
        .bottom_center => base.merge(.{ .bottom = .{ .px = 0 }, .left = .{ .percent = 50 }, .transform = .{ .translateX = .{ .percent = -50 } } }),
        .bottom_right => base.merge(.{ .bottom = .{ .px = 0 }, .right = .{ .px = 0 } }),
    };
}

fn toastStyle() zui.css.StyleRule {
    return zui.css.styles.create(.{
        .display = .flex,
        .align_items = .center,
        .justify_content = .space_between,
        .padding = .{ .px = 12 },
        .border_radius = .{ .px = 4 },
        .box_shadow = .{ .md = true },
    });
}
```

---

## JS RUNTIME INTEGRATION

```javascript
// www/zui.js - Portal handling

const ZuiPortals = {
    containers: new Map(),
    focusTrapStack: [],

    // Create or get portal container
    getContainer(target) {
        switch (target.type) {
            case 'body':
                return document.body;

            case 'id':
                return document.getElementById(target.id);

            case 'create':
                let container = document.getElementById(target.id);
                if (!container) {
                    container = document.createElement('div');
                    container.id = target.id;
                    this.getContainer(target.parent).appendChild(container);
                }
                return container;

            default:
                return document.body;
        }
    },

    // Mount portal content
    mount(portalId, target, content, options) {
        const container = this.getContainer(target);
        if (!container) {
            console.error('Portal target not found:', target);
            return;
        }

        // Create portal wrapper
        const wrapper = document.createElement('div');
        wrapper.id = `portal-${portalId}`;
        wrapper.dataset.portal = 'true';

        if (options.zIndex) {
            wrapper.style.zIndex = options.zIndex;
        }

        container.appendChild(wrapper);
        this.containers.set(portalId, wrapper);

        // Render content into wrapper
        ZuiRuntime.renderInto(wrapper, content);

        // Setup focus trap if requested
        if (options.trapFocus) {
            this.setupFocusTrap(wrapper);
        }

        // Setup click outside handler
        if (options.onClickOutside !== null) {
            this.setupClickOutside(portalId, wrapper, options.onClickOutside);
        }

        // Setup escape handler
        if (options.onEscape !== null) {
            this.setupEscapeHandler(portalId, options.onEscape);
        }

        // Store previous focus for restoration
        if (options.restoreFocus) {
            wrapper.dataset.previousFocus = document.activeElement?.id || '';
        }
    },

    // Unmount portal
    unmount(portalId) {
        const wrapper = this.containers.get(portalId);
        if (!wrapper) return;

        // Restore focus
        const previousFocusId = wrapper.dataset.previousFocus;
        if (previousFocusId) {
            const previousElement = document.getElementById(previousFocusId);
            previousElement?.focus();
        }

        // Remove focus trap
        this.removeFocusTrap(wrapper);

        // Remove event listeners
        this.cleanupListeners(portalId);

        // Remove from DOM
        wrapper.remove();
        this.containers.delete(portalId);
    },

    // Focus trap implementation
    setupFocusTrap(container) {
        const focusableSelector = 'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])';
        const focusableElements = container.querySelectorAll(focusableSelector);

        if (focusableElements.length === 0) return;

        const firstElement = focusableElements[0];
        const lastElement = focusableElements[focusableElements.length - 1];

        // Focus first element
        firstElement.focus();

        // Trap focus
        const trapHandler = (e) => {
            if (e.key !== 'Tab') return;

            if (e.shiftKey) {
                if (document.activeElement === firstElement) {
                    e.preventDefault();
                    lastElement.focus();
                }
            } else {
                if (document.activeElement === lastElement) {
                    e.preventDefault();
                    firstElement.focus();
                }
            }
        };

        container.addEventListener('keydown', trapHandler);
        container._focusTrapHandler = trapHandler;
        this.focusTrapStack.push(container);
    },

    removeFocusTrap(container) {
        if (container._focusTrapHandler) {
            container.removeEventListener('keydown', container._focusTrapHandler);
            delete container._focusTrapHandler;
        }

        const index = this.focusTrapStack.indexOf(container);
        if (index > -1) {
            this.focusTrapStack.splice(index, 1);
        }
    },

    // Click outside detection
    clickOutsideHandlers: new Map(),

    setupClickOutside(portalId, container, msgId) {
        const handler = (e) => {
            if (!container.contains(e.target)) {
                ZuiRuntime.dispatch(msgId);
            }
        };

        // Delay to avoid immediate trigger
        setTimeout(() => {
            document.addEventListener('click', handler);
            this.clickOutsideHandlers.set(portalId, handler);
        }, 0);
    },

    // Escape key handling
    escapeHandlers: new Map(),

    setupEscapeHandler(portalId, msgId) {
        const handler = (e) => {
            if (e.key === 'Escape') {
                ZuiRuntime.dispatch(msgId);
            }
        };

        document.addEventListener('keydown', handler);
        this.escapeHandlers.set(portalId, handler);
    },

    cleanupListeners(portalId) {
        const clickHandler = this.clickOutsideHandlers.get(portalId);
        if (clickHandler) {
            document.removeEventListener('click', clickHandler);
            this.clickOutsideHandlers.delete(portalId);
        }

        const escapeHandler = this.escapeHandlers.get(portalId);
        if (escapeHandler) {
            document.removeEventListener('keydown', escapeHandler);
            this.escapeHandlers.delete(portalId);
        }
    },

    // Position calculation for tooltips/dropdowns
    calculatePosition(triggerRect, position, alignment) {
        // Returns { top, left } based on trigger position
        // Implementation depends on viewport bounds checking
    },
};
```

---

## USAGE EXAMPLES

### Basic Modal

```zig
pub fn App(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return h.div(ctx, .{}, &.{
        h.button(ctx, .{
            .onClick = .open_modal,
        }, &.{
            h.text(ctx, "Open Modal"),
        }),

        Modal(Msg, ctx, .{
            .is_open = model.modal_open,
            .on_close = .close_modal,
            .title = "Confirm Action",
            .size = .medium,
            .children = &.{
                h.p(ctx, .{}, &.{
                    h.text(ctx, "Are you sure you want to proceed?"),
                }),
            },
            .footer = &.{
                h.button(ctx, .{
                    .onClick = .close_modal,
                    .class = "btn-secondary",
                }, &.{ h.text(ctx, "Cancel") }),
                h.button(ctx, .{
                    .onClick = .confirm_action,
                    .class = "btn-primary",
                }, &.{ h.text(ctx, "Confirm") }),
            },
        }),
    });
}
```

### Toast Notifications

```zig
pub fn App(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return h.div(ctx, .{}, &.{
        // Main content...

        // Toast container (always rendered, shows toasts when present)
        ToastContainer(Msg, ctx, .{
            .toasts = model.toasts,
            .position = .top_right,
            .on_dismiss = struct {
                fn f(id: u32) Msg {
                    return .{ .dismiss_toast = id };
                }
            }.f,
        }),
    });
}
```

---

## CONCLUSION

Portals provide essential functionality for UI patterns that need to break out of their container's constraints. The system integrates seamlessly with ZUI's VDOM while maintaining proper event propagation and accessibility.

**Links:**
- [← Previous: Suspense](22-suspense.md)
- [→ Next: Head Management](24-head-management.md)
- [↑ Back to index](../README.md)
