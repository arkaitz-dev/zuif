# 26 - SLOTS

**Document:** Named Slots for Component Composition
**Version:** 6.0.0
**Status:** Final
**Dependencies:** 03-virtual-dom.md, 04-html-dsl.md

---

## OVERVIEW

Slots provide a powerful composition pattern allowing parent components to inject content into specific locations within child components. This enables:

- **Flexible layouts:** Cards with headers, bodies, and footers
- **Customizable components:** Buttons with icons and text
- **Template patterns:** Layouts with named regions
- **Render props:** Passing render functions to children

---

## ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────┐
│                      SLOT SYSTEM                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Parent Component:               Child Component:            │
│  ┌─────────────────────┐        ┌─────────────────────┐     │
│  │ <Card>              │        │ Card {              │     │
│  │   <slot:header>     │───────▶│   <div class="hdr"> │     │
│  │     <h2>Title</h2>  │        │     {slots.header}  │     │
│  │   </slot:header>    │        │   </div>            │     │
│  │                     │        │                     │     │
│  │   <slot:default>    │───────▶│   <div class="body">│     │
│  │     <p>Content</p>  │        │     {slots.default} │     │
│  │   </slot:default>   │        │   </div>            │     │
│  │                     │        │                     │     │
│  │   <slot:footer>     │───────▶│   <div class="ftr"> │     │
│  │     <Button/>       │        │     {slots.footer}  │     │
│  │   </slot:footer>    │        │   </div>            │     │
│  │ </Card>             │        │ }                   │     │
│  └─────────────────────┘        └─────────────────────┘     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## SLOT TYPES

### Slots Definition

```zig
// src/slots/slots.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Slots container - holds named slot content
pub fn Slots(comptime Msg: type) type {
    return struct {
        /// Default slot content (unnamed children)
        default: []const vdom.Element(Msg) = &.{},

        /// Named slots
        named: std.StringHashMap([]const vdom.Element(Msg)),

        allocator: std.mem.Allocator,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{
                .named = std.StringHashMap([]const vdom.Element(Msg)).init(allocator),
                .allocator = allocator,
            };
        }

        pub fn deinit(self: *Self) void {
            self.named.deinit();
        }

        /// Get a named slot's content
        pub fn get(self: *const Self, name: []const u8) []const vdom.Element(Msg) {
            return self.named.get(name) orelse &.{};
        }

        /// Check if a slot has content
        pub fn has(self: *const Self, name: []const u8) bool {
            if (std.mem.eql(u8, name, "default")) {
                return self.default.len > 0;
            }
            return self.named.contains(name);
        }

        /// Get slot content or fallback
        pub fn getOr(
            self: *const Self,
            name: []const u8,
            fallback: []const vdom.Element(Msg),
        ) []const vdom.Element(Msg) {
            if (std.mem.eql(u8, name, "default")) {
                return if (self.default.len > 0) self.default else fallback;
            }
            return self.named.get(name) orelse fallback;
        }

        /// Render slot content as single element
        pub fn render(self: *const Self, name: []const u8) vdom.Element(Msg) {
            const content = self.get(name);
            return switch (content.len) {
                0 => .none,
                1 => content[0],
                else => .{ .fragment = content },
            };
        }

        /// Render slot with fallback
        pub fn renderOr(
            self: *const Self,
            name: []const u8,
            fallback: vdom.Element(Msg),
        ) vdom.Element(Msg) {
            const content = self.get(name);
            return switch (content.len) {
                0 => fallback,
                1 => content[0],
                else => .{ .fragment = content },
            };
        }
    };
}
```

### Slot Builder

```zig
// src/slots/builder.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");
const Slots = @import("slots.zig").Slots;

/// Builder for constructing slots
pub fn SlotBuilder(comptime Msg: type) type {
    return struct {
        slots: Slots(Msg),

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{
                .slots = Slots(Msg).init(allocator),
            };
        }

        /// Add default slot content
        pub fn default(self: *Self, content: []const vdom.Element(Msg)) *Self {
            self.slots.default = content;
            return self;
        }

        /// Add named slot content
        pub fn slot(self: *Self, name: []const u8, content: []const vdom.Element(Msg)) *Self {
            self.slots.named.put(name, content) catch {};
            return self;
        }

        /// Build and return the slots
        pub fn build(self: *Self) Slots(Msg) {
            return self.slots;
        }
    };
}

/// Helper to create slots from a struct
pub fn slotsFromStruct(
    comptime Msg: type,
    allocator: std.mem.Allocator,
    slot_struct: anytype,
) Slots(Msg) {
    var slots = Slots(Msg).init(allocator);

    const T = @TypeOf(slot_struct);
    const fields = std.meta.fields(T);

    inline for (fields) |field| {
        const value = @field(slot_struct, field.name);

        if (comptime std.mem.eql(u8, field.name, "default")) {
            slots.default = value;
        } else {
            slots.named.put(field.name, value) catch {};
        }
    }

    return slots;
}
```

---

## SLOT ELEMENT

### VDOM Extension

```zig
// Addition to src/vdom/element.zig

pub fn Element(comptime Msg: type) type {
    return union(enum) {
        // ... existing variants ...

        /// Slot placeholder - filled by parent
        slot: struct {
            name: []const u8,
            fallback: ?[]Element(Msg) = null,
        },

        /// Slot content - content to put in a slot
        slot_content: struct {
            name: []const u8,
            content: []Element(Msg),
        },
    };
}
```

### HTML DSL Integration

```zig
// src/html/slots.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Define a slot placeholder in a component
pub fn slot(
    comptime Msg: type,
    ctx: *AppContext,
    name: []const u8,
) vdom.Element(Msg) {
    return .{
        .slot = .{
            .name = name,
            .fallback = null,
        },
    };
}

/// Define a slot placeholder with fallback content
pub fn slotWithFallback(
    comptime Msg: type,
    ctx: *AppContext,
    name: []const u8,
    fallback: []const vdom.Element(Msg),
) vdom.Element(Msg) {
    return .{
        .slot = .{
            .name = name,
            .fallback = fallback,
        },
    };
}

/// Default slot (shorthand)
pub fn defaultSlot(
    comptime Msg: type,
    ctx: *AppContext,
) vdom.Element(Msg) {
    return slot(Msg, ctx, "default");
}

/// Provide content for a named slot
pub fn slotContent(
    comptime Msg: type,
    ctx: *AppContext,
    name: []const u8,
    content: []const vdom.Element(Msg),
) vdom.Element(Msg) {
    return .{
        .slot_content = .{
            .name = name,
            .content = content,
        },
    };
}
```

---

## COMPONENT PATTERNS

### Card Component with Slots

```zig
// src/components/card.zig
const std = @import("std");
const zui = @import("../zui.zig");
const h = zui.html;
const Element = zui.Element;
const Slots = zui.Slots;

pub fn CardProps(comptime Msg: type) type {
    return struct {
        /// Optional CSS class
        class: ?[]const u8 = null,

        /// Slots for content
        slots: Slots(Msg),

        /// Whether card is elevated
        elevated: bool = true,
    };
}

/// Card component with header, body, footer slots
pub fn Card(
    comptime Msg: type,
    ctx: *zui.AppContext,
    props: CardProps(Msg),
) Element(Msg) {
    const styles = zui.css.styles;

    return h.div(ctx, .{
        .class = props.class orelse "card",
        .style = styles.create(.{
            .background_color = .{ .var = "color-surface" },
            .border_radius = .{ .px = 8 },
            .box_shadow = if (props.elevated) .{ .md = true } else null,
            .overflow = .hidden,
        }),
    }, &.{
        // Header slot (only if content provided)
        if (props.slots.has("header"))
            h.div(ctx, .{
                .class = "card-header",
                .style = styles.create(.{
                    .padding = .{ .px = 16 },
                    .border_bottom = .{
                        .width = .{ .px = 1 },
                        .color = .{ .var = "color-border" },
                    },
                }),
            }, props.slots.get("header"))
        else
            .none,

        // Body/default slot
        h.div(ctx, .{
            .class = "card-body",
            .style = styles.create(.{
                .padding = .{ .px = 16 },
            }),
        }, props.slots.getOr("default", &.{
            h.text(ctx, "No content"),
        })),

        // Footer slot (only if content provided)
        if (props.slots.has("footer"))
            h.div(ctx, .{
                .class = "card-footer",
                .style = styles.create(.{
                    .padding = .{ .px = 16 },
                    .border_top = .{
                        .width = .{ .px = 1 },
                        .color = .{ .var = "color-border" },
                    },
                    .display = .flex,
                    .justify_content = .flex_end,
                    .gap = .{ .px = 8 },
                }),
            }, props.slots.get("footer"))
        else
            .none,
    });
}
```

### Usage with Slots

```zig
// Using the Card component
pub fn UserCard(ctx: *zui.AppContext, user: User) zui.Element(Msg) {
    return Card(Msg, ctx, .{
        .slots = zui.slotsFromStruct(Msg, ctx.frame_arena.allocator(), .{
            .header = &.{
                h.div(ctx, .{ .class = "user-header" }, &.{
                    h.img(ctx, .{
                        .src = user.avatar,
                        .class = "avatar",
                    }),
                    h.h3(ctx, .{}, &.{ h.text(ctx, user.name) }),
                }),
            },
            .default = &.{
                h.p(ctx, .{}, &.{ h.text(ctx, user.bio) }),
                h.p(ctx, .{ .class = "email" }, &.{
                    h.text(ctx, user.email),
                }),
            },
            .footer = &.{
                h.button(ctx, .{
                    .onClick = .{ .message_user = user.id },
                    .class = "btn-primary",
                }, &.{ h.text(ctx, "Message") }),
                h.button(ctx, .{
                    .onClick = .{ .follow_user = user.id },
                    .class = "btn-secondary",
                }, &.{ h.text(ctx, "Follow") }),
            },
        }),
    });
}
```

---

## LAYOUT COMPONENTS

### Two-Column Layout

```zig
// src/components/layouts.zig

pub fn TwoColumnLayout(
    comptime Msg: type,
    ctx: *zui.AppContext,
    props: struct {
        slots: Slots(Msg),
        sidebar_width: css.Length = .{ .px = 300 },
        gap: css.Length = .{ .px = 24 },
    },
) Element(Msg) {
    return h.div(ctx, .{
        .class = "two-column-layout",
        .style = zui.css.styles.create(.{
            .display = .grid,
            .grid_template_columns = .{
                .values = &.{ props.sidebar_width, .{ .fr = 1 } },
            },
            .gap = props.gap,
            .min_height = .{ .vh = 100 },
        }),
    }, &.{
        // Sidebar slot
        h.aside(ctx, .{ .class = "sidebar" }, props.slots.get("sidebar")),

        // Main content slot
        h.main(ctx, .{ .class = "main-content" }, props.slots.get("default")),
    });
}

/// Three-section layout (header, main, footer)
pub fn AppLayout(
    comptime Msg: type,
    ctx: *zui.AppContext,
    slots: Slots(Msg),
) Element(Msg) {
    return h.div(ctx, .{
        .class = "app-layout",
        .style = zui.css.styles.create(.{
            .display = .flex,
            .flex_direction = .column,
            .min_height = .{ .vh = 100 },
        }),
    }, &.{
        // Header
        h.header(ctx, .{
            .style = zui.css.styles.create(.{
                .flex_shrink = 0,
            }),
        }, slots.getOr("header", &.{})),

        // Main content (grows to fill space)
        h.main(ctx, .{
            .style = zui.css.styles.create(.{
                .flex = .{ .grow = 1 },
            }),
        }, slots.get("default")),

        // Footer
        h.footer(ctx, .{
            .style = zui.css.styles.create(.{
                .flex_shrink = 0,
            }),
        }, slots.getOr("footer", &.{})),
    });
}
```

### Usage

```zig
pub fn App(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return AppLayout(Msg, ctx, zui.slotsFromStruct(Msg, ctx.frame_arena.allocator(), .{
        .header = &.{
            NavBar(ctx, model),
        },
        .default = &.{
            Router(ctx, model),
        },
        .footer = &.{
            Footer(ctx),
        },
    }));
}
```

---

## SCOPED SLOTS (RENDER PROPS)

### Definition

Scoped slots allow child components to pass data back to the slot content, enabling render prop patterns.

```zig
// src/slots/scoped.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Scoped slot - passes data to the slot renderer
pub fn ScopedSlot(comptime Msg: type, comptime SlotData: type) type {
    return struct {
        /// Render function that receives slot data
        render: *const fn (SlotData, *AppContext) vdom.Element(Msg),
    };
}

/// Scoped slots container
pub fn ScopedSlots(comptime Msg: type) type {
    return struct {
        /// Store scoped slot renderers
        renderers: std.StringHashMap(ScopedRenderer),
        allocator: std.mem.Allocator,

        const ScopedRenderer = struct {
            render_fn: *const anyopaque,
            type_id: usize,
        };

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{
                .renderers = std.StringHashMap(ScopedRenderer).init(allocator),
                .allocator = allocator,
            };
        }

        /// Register a scoped slot renderer
        pub fn register(
            self: *Self,
            comptime SlotData: type,
            name: []const u8,
            render: *const fn (SlotData, *AppContext) vdom.Element(Msg),
        ) void {
            self.renderers.put(name, .{
                .render_fn = @ptrCast(render),
                .type_id = @typeInfo(SlotData).hash,
            }) catch {};
        }

        /// Render scoped slot with data
        pub fn render(
            self: *const Self,
            comptime SlotData: type,
            name: []const u8,
            data: SlotData,
            ctx: *AppContext,
        ) vdom.Element(Msg) {
            const renderer = self.renderers.get(name) orelse return .none;

            const render_fn: *const fn (SlotData, *AppContext) vdom.Element(Msg) =
                @ptrCast(@alignCast(renderer.render_fn));

            return render_fn(data, ctx);
        }
    };
}
```

### List Component with Scoped Slot

```zig
// src/components/data_list.zig

pub fn DataListProps(comptime Msg: type, comptime T: type) type {
    return struct {
        items: []const T,
        /// Render function for each item
        render_item: *const fn (T, usize, *AppContext) vdom.Element(Msg),
        /// Optional empty state
        empty: ?[]const vdom.Element(Msg) = null,
        /// Optional loading state
        loading: bool = false,
        loading_content: ?[]const vdom.Element(Msg) = null,
    };
}

/// Generic data list with scoped slots for items
pub fn DataList(
    comptime Msg: type,
    comptime T: type,
    ctx: *zui.AppContext,
    props: DataListProps(Msg, T),
) Element(Msg) {
    // Loading state
    if (props.loading) {
        return h.div(ctx, .{ .class = "data-list-loading" },
            props.loading_content orelse &.{
                h.text(ctx, "Loading..."),
            });
    }

    // Empty state
    if (props.items.len == 0) {
        return h.div(ctx, .{ .class = "data-list-empty" },
            props.empty orelse &.{
                h.text(ctx, "No items"),
            });
    }

    // Render items
    var children = ctx.frame_arena.allocator().alloc(
        Element(Msg),
        props.items.len,
    ) catch return .none;

    for (props.items, 0..) |item, index| {
        children[index] = props.render_item(item, index, ctx);
    }

    return h.ul(ctx, .{ .class = "data-list" }, children);
}
```

### Usage with Scoped Slots

```zig
pub fn UserList(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return DataList(Msg, User, ctx, .{
        .items = model.users,
        .render_item = struct {
            fn render(user: User, index: usize, c: *zui.AppContext) zui.Element(Msg) {
                return h.li(ctx, .{
                    .key = user.id,
                    .class = if (index % 2 == 0) "even" else "odd",
                }, &.{
                    h.img(c, .{ .src = user.avatar, .class = "avatar-sm" }),
                    h.span(c, .{}, &.{ h.text(c, user.name) }),
                    h.button(c, .{
                        .onClick = .{ .select_user = user.id },
                    }, &.{ h.text(c, "Select") }),
                });
            }
        }.render,
        .empty = &.{
            h.div(ctx, .{ .class = "empty-state" }, &.{
                h.p(ctx, .{}, &.{ h.text(ctx, "No users found") }),
                h.button(ctx, .{ .onClick = .refresh_users }, &.{
                    h.text(ctx, "Refresh"),
                }),
            }),
        },
    });
}
```

---

## TABLE COMPONENT WITH COLUMN SLOTS

```zig
// src/components/data_table.zig

pub fn Column(comptime Msg: type, comptime T: type) type {
    return struct {
        /// Column key (for sorting, etc.)
        key: []const u8,
        /// Header content
        header: []const u8,
        /// Cell renderer
        render: *const fn (T, *AppContext) vdom.Element(Msg),
        /// Optional width
        width: ?css.Length = null,
        /// Sortable
        sortable: bool = false,
    };
}

pub fn DataTableProps(comptime Msg: type, comptime T: type) type {
    return struct {
        items: []const T,
        columns: []const Column(Msg, T),
        /// Row key extractor
        row_key: *const fn (T) []const u8,
        /// Sort state
        sort_column: ?[]const u8 = null,
        sort_direction: enum { asc, desc } = .asc,
        on_sort: ?*const fn ([]const u8) Msg = null,
        /// Selection
        selectable: bool = false,
        selected_keys: []const []const u8 = &.{},
        on_select: ?*const fn ([]const u8) Msg = null,
    };
}

pub fn DataTable(
    comptime Msg: type,
    comptime T: type,
    ctx: *zui.AppContext,
    props: DataTableProps(Msg, T),
) Element(Msg) {
    const allocator = ctx.frame_arena.allocator();

    // Build header row
    var header_cells = allocator.alloc(Element(Msg), props.columns.len) catch return .none;
    for (props.columns, 0..) |col, i| {
        header_cells[i] = h.th(ctx, .{
            .class = if (col.sortable) "sortable" else null,
            .onClick = if (col.sortable and props.on_sort != null)
                props.on_sort.?(col.key)
            else
                null,
            .style = if (col.width) |w|
                zui.css.styles.create(.{ .width = w })
            else
                null,
        }, &.{
            h.text(ctx, col.header),
            if (props.sort_column != null and
                std.mem.eql(u8, props.sort_column.?, col.key))
                h.span(ctx, .{ .class = "sort-indicator" }, &.{
                    h.text(ctx, if (props.sort_direction == .asc) "↑" else "↓"),
                })
            else
                .none,
        });
    }

    // Build body rows
    var body_rows = allocator.alloc(Element(Msg), props.items.len) catch return .none;
    for (props.items, 0..) |item, row_idx| {
        const row_key = props.row_key(item);

        var cells = allocator.alloc(Element(Msg), props.columns.len) catch continue;
        for (props.columns, 0..) |col, col_idx| {
            cells[col_idx] = h.td(ctx, .{}, &.{
                col.render(item, ctx),
            });
        }

        body_rows[row_idx] = h.tr(ctx, .{
            .key = row_key,
            .class = if (isSelected(row_key, props.selected_keys)) "selected" else null,
            .onClick = if (props.selectable and props.on_select != null)
                props.on_select.?(row_key)
            else
                null,
        }, cells);
    }

    return h.table(ctx, .{ .class = "data-table" }, &.{
        h.thead(ctx, .{}, &.{
            h.tr(ctx, .{}, header_cells),
        }),
        h.tbody(ctx, .{}, body_rows),
    });
}

fn isSelected(key: []const u8, selected: []const []const u8) bool {
    for (selected) |s| {
        if (std.mem.eql(u8, s, key)) return true;
    }
    return false;
}
```

### Usage

```zig
pub fn UsersTable(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return DataTable(Msg, User, ctx, .{
        .items = model.users,
        .row_key = struct {
            fn f(user: User) []const u8 {
                return user.id;
            }
        }.f,
        .columns = &.{
            .{
                .key = "avatar",
                .header = "",
                .width = .{ .px = 50 },
                .render = struct {
                    fn f(user: User, c: *zui.AppContext) zui.Element(Msg) {
                        return h.img(c, .{
                            .src = user.avatar,
                            .class = "avatar-sm",
                        });
                    }
                }.f,
            },
            .{
                .key = "name",
                .header = "Name",
                .sortable = true,
                .render = struct {
                    fn f(user: User, c: *zui.AppContext) zui.Element(Msg) {
                        return h.text(c, user.name);
                    }
                }.f,
            },
            .{
                .key = "email",
                .header = "Email",
                .sortable = true,
                .render = struct {
                    fn f(user: User, c: *zui.AppContext) zui.Element(Msg) {
                        return h.a(c, .{ .href = "mailto:" ++ user.email }, &.{
                            h.text(c, user.email),
                        });
                    }
                }.f,
            },
            .{
                .key = "actions",
                .header = "Actions",
                .width = .{ .px = 120 },
                .render = struct {
                    fn f(user: User, c: *zui.AppContext) zui.Element(Msg) {
                        return h.div(c, .{ .class = "actions" }, &.{
                            h.button(c, .{
                                .onClick = .{ .edit_user = user.id },
                            }, &.{ h.text(c, "Edit") }),
                            h.button(c, .{
                                .onClick = .{ .delete_user = user.id },
                                .class = "danger",
                            }, &.{ h.text(c, "Delete") }),
                        });
                    }
                }.f,
            },
        },
        .sort_column = model.sort_column,
        .sort_direction = model.sort_direction,
        .on_sort = struct {
            fn f(col: []const u8) Msg {
                return .{ .sort_by = col };
            }
        }.f,
        .selectable = true,
        .selected_keys = model.selected_users,
        .on_select = struct {
            fn f(key: []const u8) Msg {
                return .{ .toggle_user_selection = key };
            }
        }.f,
    });
}
```

---

## DIALOG WITH ACTION SLOTS

```zig
// src/components/dialog.zig

pub fn DialogProps(comptime Msg: type) type {
    return struct {
        is_open: bool,
        on_close: Msg,
        title: ?[]const u8 = null,
        slots: Slots(Msg),
        size: DialogSize = .medium,
    };
}

pub const DialogSize = enum { small, medium, large, full };

pub fn Dialog(
    comptime Msg: type,
    ctx: *zui.AppContext,
    props: DialogProps(Msg),
) Element(Msg) {
    if (!props.is_open) return .none;

    return h.portal(Msg, ctx, .{
        .target = .body,
        .trap_focus = true,
        .on_escape = props.on_close,
        .children = &.{
            // Backdrop
            h.div(ctx, .{
                .class = "dialog-backdrop",
                .onClick = props.on_close,
            }, &.{}),

            // Dialog container
            h.div(ctx, .{
                .class = "dialog",
                .attrs = &.{
                    .{ "role", "dialog" },
                    .{ "aria-modal", "true" },
                },
            }, &.{
                // Header (with title slot or default)
                h.div(ctx, .{ .class = "dialog-header" },
                    props.slots.getOr("header", &.{
                        if (props.title) |t|
                            h.h2(ctx, .{}, &.{ h.text(ctx, t) })
                        else
                            .none,
                        h.button(ctx, .{
                            .class = "dialog-close",
                            .onClick = props.on_close,
                            .attrs = &.{ .{ "aria-label", "Close" } },
                        }, &.{ h.text(ctx, "×") }),
                    })),

                // Body
                h.div(ctx, .{ .class = "dialog-body" },
                    props.slots.get("default")),

                // Footer with actions
                if (props.slots.has("actions"))
                    h.div(ctx, .{ .class = "dialog-actions" },
                        props.slots.get("actions"))
                else
                    .none,
            }),
        },
    });
}
```

---

## CONDITIONAL SLOTS

```zig
/// Render slot only if condition is true
pub fn conditionalSlot(
    comptime Msg: type,
    slots: *const Slots(Msg),
    name: []const u8,
    condition: bool,
) vdom.Element(Msg) {
    if (!condition) return .none;
    return slots.render(name);
}

/// Render one of two slots based on condition
pub fn eitherSlot(
    comptime Msg: type,
    slots: *const Slots(Msg),
    condition: bool,
    true_slot: []const u8,
    false_slot: []const u8,
) vdom.Element(Msg) {
    return if (condition)
        slots.render(true_slot)
    else
        slots.render(false_slot);
}
```

---

## CONCLUSION

The slots system provides a powerful and flexible way to compose components in ZUI. From simple content injection to complex scoped slots with render props, developers have full control over component composition while maintaining type safety.

**Links:**
- [← Previous: Transitions](25-transitions.md)
- [→ Next: Spin SDK](27-spin-sdk.md)
- [↑ Back to index](../README.md)
