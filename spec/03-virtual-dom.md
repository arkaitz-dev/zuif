# 03 - VIRTUAL DOM

**Document:** Complete Virtual DOM with Diffing and Reconciliation
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

ZUI's Virtual DOM is an in-memory representation of the DOM tree that enables efficient updates through diffing and reconciliation. It implements an O(n) algorithm inspired by React Fiber and Elm.

---

## ARCHITECTURE

```
┌─────────────────────────────────────────────────────────┐
│                    VIRTUAL DOM                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐      ┌─────────────┐                  │
│  │   Element   │─────▶│   Diffing   │                  │
│  │    Tree     │      │  Algorithm  │                  │
│  └─────────────┘      └──────┬──────┘                  │
│                              │                          │
│                              ▼                          │
│                       ┌─────────────┐                   │
│                       │   Patches   │                   │
│                       └──────┬──────┘                   │
│                              │                          │
│                              ▼                          │
│                    ┌──────────────────┐                 │
│                    │  Reconciliation  │                 │
│                    │   (DOM Updates)  │                 │
│                    └──────────────────┘                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## ELEMENT TYPES

### Core Element Definition

```zig
// src/vdom/element.zig
const std = @import("std");
const Attrs = @import("../html/attrs.zig").Attrs;

/// Virtual DOM element - parameterized by message type
pub fn Element(comptime Msg: type) type {
    return union(enum) {
        /// Regular HTML element
        element: struct {
            tag: []const u8,
            key: ?[]const u8,
            attrs: Attrs(Msg),
            children: []Element(Msg),
        },

        /// Text node
        text: []const u8,

        /// Empty node (no rendering)
        none,

        /// Keyed list for efficient reordering
        keyed: struct {
            children: []struct {
                key: []const u8,
                element: *Element(Msg),
            },
        },

        /// Lazy evaluation for performance
        lazy: struct {
            id: []const u8,
            thunk: *const fn (*AppContext) Element(Msg),
        },

        /// Map messages to parent type
        mapped: struct {
            child: *Element(ChildMsg),
            mapper: *const fn (ChildMsg) Msg,

            const ChildMsg = blk: {
                // Extract child message type from mapper
                const mapper_info = @typeInfo(@TypeOf(mapper));
                const fn_info = mapper_info.Fn;
                const param_type = fn_info.params[0].type.?;
                break :blk param_type;
            };
        },
    };
}
```

### Element Constructors

```zig
/// Create an element node
pub fn element(
    comptime Msg: type,
    tag: []const u8,
    key: ?[]const u8,
    attrs: Attrs(Msg),
    children: []Element(Msg),
) Element(Msg) {
    return .{
        .element = .{
            .tag = tag,
            .key = key,
            .attrs = attrs,
            .children = children,
        },
    };
}

/// Create a text node
pub fn text(comptime Msg: type, content: []const u8) Element(Msg) {
    return .{ .text = content };
}

/// Create empty node
pub fn none(comptime Msg: type) Element(Msg) {
    return .none;
}

/// Create keyed list
pub fn keyed(
    comptime Msg: type,
    items: []struct {
        key: []const u8,
        element: Element(Msg),
    },
) Element(Msg) {
    return .{
        .keyed = .{
            .children = items,
        },
    };
}
```

---

## DIFFING ALGORITHM

### Diff Types

```zig
// src/vdom/diff.zig
const std = @import("std");
const Element = @import("element.zig").Element;

/// Patch operations for DOM updates
pub fn Patch(comptime Msg: type) type {
    return union(enum) {
        /// Create new node
        create: struct {
            parent_id: u32,
            element: Element(Msg),
        },

        /// Remove node
        remove: struct {
            node_id: u32,
        },

        /// Replace node
        replace: struct {
            node_id: u32,
            new_element: Element(Msg),
        },

        /// Update attributes
        update_attrs: struct {
            node_id: u32,
            added: []struct {
                key: []const u8,
                value: []const u8,
            },
            removed: [][]const u8,
            updated: []struct {
                key: []const u8,
                old_value: []const u8,
                new_value: []const u8,
            },
        },

        /// Update text content
        update_text: struct {
            node_id: u32,
            old_text: []const u8,
            new_text: []const u8,
        },

        /// Reorder children (for keyed lists)
        reorder: struct {
            parent_id: u32,
            moves: []struct {
                from_index: u32,
                to_index: u32,
            },
        },
    };
}
```

### Main Diff Function

```zig
/// Diff two virtual DOM trees
pub fn diff(
    comptime Msg: type,
    allocator: std.mem.Allocator,
    old: ?Element(Msg),
    new: Element(Msg),
    node_id: u32,
) ![]Patch(Msg) {
    var patches = std.ArrayList(Patch(Msg)).init(allocator);

    // Both null - no changes
    if (old == null and new == .none) {
        return patches.toOwnedSlice();
    }

    // Old is null - create new
    if (old == null) {
        try patches.append(.{
            .create = .{
                .parent_id = node_id,
                .element = new,
            },
        });
        return patches.toOwnedSlice();
    }

    const old_val = old.?;

    // New is none - remove old
    if (new == .none) {
        try patches.append(.{
            .remove = .{ .node_id = node_id },
        });
        return patches.toOwnedSlice();
    }

    // Different types - replace
    if (std.meta.activeTag(old_val) != std.meta.activeTag(new)) {
        try patches.append(.{
            .replace = .{
                .node_id = node_id,
                .new_element = new,
            },
        });
        return patches.toOwnedSlice();
    }

    // Same type - diff contents
    switch (new) {
        .text => |new_text| {
            const old_text = old_val.text;
            if (!std.mem.eql(u8, old_text, new_text)) {
                try patches.append(.{
                    .update_text = .{
                        .node_id = node_id,
                        .old_text = old_text,
                        .new_text = new_text,
                    },
                });
            }
        },

        .element => |new_el| {
            const old_el = old_val.element;

            // Different tags - replace
            if (!std.mem.eql(u8, old_el.tag, new_el.tag)) {
                try patches.append(.{
                    .replace = .{
                        .node_id = node_id,
                        .new_element = new,
                    },
                });
                return patches.toOwnedSlice();
            }

            // Diff attributes
            try diffAttrs(Msg, &patches, node_id, old_el.attrs, new_el.attrs);

            // Diff children
            if (new_el.key != null or old_el.key != null) {
                // Keyed children
                try diffKeyedChildren(Msg, allocator, &patches, node_id, old_el.children, new_el.children);
            } else {
                // Non-keyed children
                try diffChildren(Msg, allocator, &patches, node_id, old_el.children, new_el.children);
            }
        },

        .keyed => |new_keyed| {
            const old_keyed = old_val.keyed;
            try diffKeyedList(Msg, allocator, &patches, node_id, old_keyed.children, new_keyed.children);
        },

        .none => {},

        .lazy => {
            // Lazy nodes are evaluated before diffing
            unreachable;
        },

        .mapped => {
            // Mapped nodes are unwrapped before diffing
            unreachable;
        },
    }

    return patches.toOwnedSlice();
}
```

### Diff Attributes

```zig
fn diffAttrs(
    comptime Msg: type,
    patches: *std.ArrayList(Patch(Msg)),
    node_id: u32,
    old_attrs: Attrs(Msg),
    new_attrs: Attrs(Msg),
) !void {
    var added = std.ArrayList(struct {
        key: []const u8,
        value: []const u8,
    }).init(patches.allocator);

    var removed = std.ArrayList([]const u8).init(patches.allocator);

    var updated = std.ArrayList(struct {
        key: []const u8,
        old_value: []const u8,
        new_value: []const u8,
    }).init(patches.allocator);

    // Find added and updated
    var new_iter = new_attrs.iterator();
    while (new_iter.next()) |new_attr| {
        if (old_attrs.get(new_attr.key)) |old_value| {
            if (!std.mem.eql(u8, old_value, new_attr.value)) {
                try updated.append(.{
                    .key = new_attr.key,
                    .old_value = old_value,
                    .new_value = new_attr.value,
                });
            }
        } else {
            try added.append(.{
                .key = new_attr.key,
                .value = new_attr.value,
            });
        }
    }

    // Find removed
    var old_iter = old_attrs.iterator();
    while (old_iter.next()) |old_attr| {
        if (!new_attrs.has(old_attr.key)) {
            try removed.append(old_attr.key);
        }
    }

    // Add patch if there are changes
    if (added.items.len > 0 or removed.items.len > 0 or updated.items.len > 0) {
        try patches.append(.{
            .update_attrs = .{
                .node_id = node_id,
                .added = try added.toOwnedSlice(),
                .removed = try removed.toOwnedSlice(),
                .updated = try updated.toOwnedSlice(),
            },
        });
    }
}
```

### Diff Children (Non-Keyed)

```zig
fn diffChildren(
    comptime Msg: type,
    allocator: std.mem.Allocator,
    patches: *std.ArrayList(Patch(Msg)),
    parent_id: u32,
    old_children: []Element(Msg),
    new_children: []Element(Msg),
) !void {
    const min_len = @min(old_children.len, new_children.len);

    // Diff existing children
    for (0..min_len) |i| {
        const child_id = parent_id * 1000 + @as(u32, @intCast(i));
        const child_patches = try diff(Msg, allocator, old_children[i], new_children[i], child_id);
        try patches.appendSlice(child_patches);
    }

    // Remove extra old children
    if (old_children.len > new_children.len) {
        for (min_len..old_children.len) |i| {
            const child_id = parent_id * 1000 + @as(u32, @intCast(i));
            try patches.append(.{
                .remove = .{ .node_id = child_id },
            });
        }
    }

    // Add extra new children
    if (new_children.len > old_children.len) {
        for (min_len..new_children.len) |i| {
            try patches.append(.{
                .create = .{
                    .parent_id = parent_id,
                    .element = new_children[i],
                },
            });
        }
    }
}
```

### Diff Keyed Children

```zig
fn diffKeyedChildren(
    comptime Msg: type,
    allocator: std.mem.Allocator,
    patches: *std.ArrayList(Patch(Msg)),
    parent_id: u32,
    old_children: []Element(Msg),
    new_children: []Element(Msg),
) !void {
    // Build key maps
    var old_keys = std.StringHashMap(usize).init(allocator);
    defer old_keys.deinit();

    var new_keys = std.StringHashMap(usize).init(allocator);
    defer new_keys.deinit();

    for (old_children, 0..) |child, i| {
        if (child.element.key) |key| {
            try old_keys.put(key, i);
        }
    }

    for (new_children, 0..) |child, i| {
        if (child.element.key) |key| {
            try new_keys.put(key, i);
        }
    }

    var moves = std.ArrayList(struct {
        from_index: u32,
        to_index: u32,
    }).init(allocator);

    // Find moves and updates
    for (new_children, 0..) |new_child, new_idx| {
        const key = new_child.element.key orelse continue;

        if (old_keys.get(key)) |old_idx| {
            // Child exists, check if moved
            if (old_idx != new_idx) {
                try moves.append(.{
                    .from_index = @intCast(old_idx),
                    .to_index = @intCast(new_idx),
                });
            }

            // Diff the child
            const child_id = parent_id * 1000 + @as(u32, @intCast(new_idx));
            const child_patches = try diff(
                Msg,
                allocator,
                old_children[old_idx],
                new_child,
                child_id,
            );
            try patches.appendSlice(child_patches);
        } else {
            // New child
            try patches.append(.{
                .create = .{
                    .parent_id = parent_id,
                    .element = new_child,
                },
            });
        }
    }

    // Find removed children
    for (old_children) |old_child| {
        const key = old_child.element.key orelse continue;

        if (!new_keys.contains(key)) {
            const old_idx = old_keys.get(key).?;
            const child_id = parent_id * 1000 + @as(u32, @intCast(old_idx));
            try patches.append(.{
                .remove = .{ .node_id = child_id },
            });
        }
    }

    // Add reorder patch if needed
    if (moves.items.len > 0) {
        try patches.append(.{
            .reorder = .{
                .parent_id = parent_id,
                .moves = try moves.toOwnedSlice(),
            },
        });
    }
}
```

---

## RECONCILIATION

### Patch Application

```zig
// src/vdom/reconciler.zig
const std = @import("std");
const Patch = @import("diff.zig").Patch;

/// Apply patches to update the DOM
pub fn reconcile(
    comptime Msg: type,
    patches: []Patch(Msg),
) !void {
    for (patches) |patch| {
        try applyPatch(Msg, patch);
    }
}

fn applyPatch(comptime Msg: type, patch: Patch(Msg)) !void {
    switch (patch) {
        .create => |create| {
            try createNode(Msg, create.parent_id, create.element);
        },

        .remove => |remove| {
            try removeNode(remove.node_id);
        },

        .replace => |replace| {
            try replaceNode(Msg, replace.node_id, replace.new_element);
        },

        .update_attrs => |update| {
            try updateAttrs(update.node_id, update.added, update.removed, update.updated);
        },

        .update_text => |update| {
            try updateText(update.node_id, update.new_text);
        },

        .reorder => |reorder| {
            try reorderChildren(reorder.parent_id, reorder.moves);
        },
    }
}
```

### WASM Exports for DOM Operations

```zig
// These call into JS runtime via WASM imports
extern "zui" fn js_createElement(tag_ptr: [*]const u8, tag_len: usize) u32;
extern "zui" fn js_createTextNode(text_ptr: [*]const u8, text_len: usize) u32;
extern "zui" fn js_appendChild(parent_id: u32, child_id: u32) void;
extern "zui" fn js_removeChild(parent_id: u32, child_id: u32) void;
extern "zui" fn js_replaceChild(parent_id: u32, old_id: u32, new_id: u32) void;
extern "zui" fn js_setAttribute(node_id: u32, key_ptr: [*]const u8, key_len: usize, val_ptr: [*]const u8, val_len: usize) void;
extern "zui" fn js_removeAttribute(node_id: u32, key_ptr: [*]const u8, key_len: usize) void;
extern "zui" fn js_setTextContent(node_id: u32, text_ptr: [*]const u8, text_len: usize) void;
extern "zui" fn js_insertBefore(parent_id: u32, new_id: u32, ref_id: u32) void;

fn createNode(comptime Msg: type, parent_id: u32, element: Element(Msg)) !void {
    switch (element) {
        .text => |txt| {
            const node_id = js_createTextNode(txt.ptr, txt.len);
            js_appendChild(parent_id, node_id);
        },

        .element => |el| {
            const node_id = js_createElement(el.tag.ptr, el.tag.len);

            // Set attributes
            var iter = el.attrs.iterator();
            while (iter.next()) |attr| {
                js_setAttribute(node_id, attr.key.ptr, attr.key.len, attr.value.ptr, attr.value.len);
            }

            // Append to parent
            js_appendChild(parent_id, node_id);

            // Create children
            for (el.children) |child| {
                try createNode(Msg, node_id, child);
            }
        },

        .none => {},

        else => unreachable,
    }
}
```

---

## OPTIMIZATION TECHNIQUES

### 1. Key-Based Reconciliation

```zig
// Efficient list rendering with keys
pub fn UserList(ctx: *AppContext, users: []User) Element(Msg) {
    const items = try ctx.allocator.alloc(struct {
        key: []const u8,
        element: Element(Msg),
    }, users.len);

    for (users, 0..) |user, i| {
        items[i] = .{
            .key = user.id,
            .element = UserCard(ctx, user),
        };
    }

    return Element(Msg).keyed(items);
}
```

### 2. Lazy Evaluation

```zig
// Defer rendering expensive components
pub fn ExpensiveComponent(ctx: *AppContext, data: *const Data) Element(Msg) {
    return Element(Msg).lazy(.{
        .id = "expensive-component",
        .thunk = struct {
            fn render(c: *AppContext) Element(Msg) {
                // Only evaluated if needed
                return computeExpensiveView(c, data);
            }
        }.render,
    });
}
```

### 3. Shouldupdate Pattern

```zig
// Memoization at component level
pub fn MemoizedComponent(
    ctx: *AppContext,
    props: Props,
    last_props: ?Props,
) Element(Msg) {
    if (last_props) |last| {
        if (std.meta.eql(props, last)) {
            // Return cached element
            return cached_element;
        }
    }

    // Render and cache
    const element = render(ctx, props);
    cached_element = element;
    return element;
}
```

---

## CONCLUSION

ZUI's Virtual DOM provides efficient DOM updates through O(n) diffing and intelligent reconciliation. It supports keys for lists, lazy evaluation, and message mapping for component composition.

**Links:**
- [← Previous: Context System](02-context-system.md)
- [→ Next: HTML DSL](04-html-dsl.md)
- [↑ Back to index](../README.md)
