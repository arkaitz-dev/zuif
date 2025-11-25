# 16 - EXAMPLES

**Document:** Complete Application Examples
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

This document provides complete and functional examples of ZUI applications, from simple counters to applications with routing and HTTP.

---

## EXAMPLE 1: COUNTER

### Complete Implementation

```zig
// examples/counter/src/main.zig
const std = @import("std");
const zui = @import("zui");

pub const Model = struct {
    count: i32 = 0,
};

pub const Msg = enum {
    increment,
    decrement,
    reset,
};

pub fn init() Model {
    return .{};
}

pub fn update(model: *Model, msg: Msg, ctx: *zui.AppContext) zui.Effect(Msg) {
    _ = ctx;
    
    switch (msg) {
        .increment => model.count += 1,
        .decrement => model.count -= 1,
        .reset => model.count = 0,
    }
    
    return zui.Effect.none;
}

pub fn view(model: *const Model, ctx: *zui.AppContext) zui.Element(Msg) {
    const styles = zui.css.styles;
    const h = zui.h;
    
    return h.div(ctx, Msg, .{
        .style = styles.create(.{
            .display = .flex,
            .flex_direction = .column,
            .align_items = .center,
            .justify_content = .center,
            .min_height = .{ .vh = 100 },
            .gap = .{ .rem = 2 },
            .background_color = ctx.theme.colors.background,
        }),
    }, &.{
        h.h1(ctx, Msg, .{
            .style = styles.create(.{
                .font_size = .{ .rem = 3 },
                .color = ctx.theme.colors.text,
            }),
        }, &.{
            h.text(ctx, Msg, "Counter"),
        }),
        
        h.div(ctx, Msg, .{
            .style = styles.create(.{
                .font_size = .{ .rem = 4 },
                .font_weight = .{ .number = 700 },
                .color = ctx.theme.colors.primary,
            }),
        }, &.{
            h.textFmt(ctx, Msg, "{d}", .{model.count}),
        }),
        
        h.div(ctx, Msg, .{
            .style = styles.create(.{
                .display = .flex,
                .gap = .{ .rem = 1 },
            }),
        }, &.{
            h.button(ctx, Msg, .{
                .onClick = .decrement,
                .style = styles.create(styles.button.secondary()),
            }, &.{
                h.text(ctx, Msg, "-"),
            }),
            
            h.button(ctx, Msg, .{
                .onClick = .reset,
                .style = styles.create(styles.button.secondary()),
            }, &.{
                h.text(ctx, Msg, "Reset"),
            }),
            
            h.button(ctx, Msg, .{
                .onClick = .increment,
                .style = styles.create(styles.button.primary()),
            }, &.{
                h.text(ctx, Msg, "+"),
            }),
        }),
    });
}

pub fn main() void {
    zui.app.run(.{
        .init = init,
        .update = update,
        .view = view,
    }, .{});
}
```

---

## EXAMPLE 2: TODO APP

### Complete Implementation

```zig
// examples/todo/src/main.zig
const std = @import("std");
const zui = @import("zui");

pub const Model = struct {
    todos: std.ArrayList(Todo),
    input: []const u8,
    filter: Filter,
    next_id: u32,
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) !Model {
        return .{
            .todos = std.ArrayList(Todo).init(allocator),
            .input = "",
            .filter = .all,
            .next_id = 1,
            .allocator = allocator,
        };
    }
    
    pub fn deinit(self: *Model) void {
        self.todos.deinit();
    }
};

pub const Todo = struct {
    id: u32,
    text: []const u8,
    completed: bool,
};

pub const Filter = enum {
    all,
    active,
    completed,
};

pub const Msg = union(enum) {
    input_changed: []const u8,
    add_todo,
    toggle_todo: u32,
    delete_todo: u32,
    set_filter: Filter,
    clear_completed,
};

pub fn update(model: *Model, msg: Msg, ctx: *zui.AppContext) zui.Effect(Msg) {
    _ = ctx;
    
    switch (msg) {
        .input_changed => |text| {
            model.input = text;
        },
        
        .add_todo => {
            if (model.input.len > 0) {
                model.todos.append(.{
                    .id = model.next_id,
                    .text = model.input,
                    .completed = false,
                }) catch {};
                model.next_id += 1;
                model.input = "";
            }
        },
        
        .toggle_todo => |id| {
            for (model.todos.items) |*todo| {
                if (todo.id == id) {
                    todo.completed = !todo.completed;
                    break;
                }
            }
        },
        
        .delete_todo => |id| {
            var i: usize = 0;
            while (i < model.todos.items.len) {
                if (model.todos.items[i].id == id) {
                    _ = model.todos.orderedRemove(i);
                    break;
                }
                i += 1;
            }
        },
        
        .set_filter => |filter| {
            model.filter = filter;
        },
        
        .clear_completed => {
            var i: usize = 0;
            while (i < model.todos.items.len) {
                if (model.todos.items[i].completed) {
                    _ = model.todos.orderedRemove(i);
                } else {
                    i += 1;
                }
            }
        },
    }
    
    return zui.Effect.none;
}

pub fn view(model: *const Model, ctx: *zui.AppContext) zui.Element(Msg) {
    const styles = zui.css.styles;
    const h = zui.h;
    
    return h.div(ctx, Msg, .{
        .style = styles.create(.{
            .max_width = .{ .px = 600 },
            .margin = .auto,
            .padding = .{ .rem = 2 },
        }),
    }, &.{
        h.h1(ctx, Msg, .{
            .style = styles.create(styles.text.heading()),
        }, &.{
            h.text(ctx, Msg, "todos"),
        }),
        
        // Input
        h.div(ctx, Msg, .{
            .style = styles.create(.{
                .display = .flex,
                .gap = .{ .rem = 0.5 },
                .margin_bottom = .{ .rem = 1 },
            }),
        }, &.{
            h.input(ctx, Msg, .{
                .type = "text",
                .value = model.input,
                .placeholder = "What needs to be done?",
                .onInput = .{ .input_changed = "" }, // would get actual value
                .style = styles.create(.{
                    .flex_grow = 1,
                    .padding = .{ .rem = 0.5 },
                }),
            }),
            
            h.button(ctx, Msg, .{
                .onClick = .add_todo,
                .style = styles.create(styles.button.primary()),
            }, &.{
                h.text(ctx, Msg, "Add"),
            }),
        }),
        
        // Todo list
        viewTodoList(model, ctx),
        
        // Footer
        viewFooter(model, ctx),
    });
}

fn viewTodoList(model: *const Model, ctx: *zui.AppContext) zui.Element(Msg) {
    const h = zui.h;
    const styles = zui.css.styles;
    
    var filtered = std.ArrayList(zui.Element(Msg)).init(ctx.allocator);
    
    for (model.todos.items) |todo| {
        const show = switch (model.filter) {
            .all => true,
            .active => !todo.completed,
            .completed => todo.completed,
        };
        
        if (show) {
            filtered.append(viewTodoItem(todo, ctx)) catch {};
        }
    }
    
    return h.ul(ctx, Msg, .{
        .style = styles.create(.{
            .list_style = .none,
            .padding = .zero,
        }),
    }, filtered.toOwnedSlice() catch &.{});
}

fn viewTodoItem(todo: Todo, ctx: *zui.AppContext) zui.Element(Msg) {
    const h = zui.h;
    const styles = zui.css.styles;
    
    return h.li(ctx, Msg, .{
        .key = try std.fmt.allocPrint(ctx.allocator, "{d}", .{todo.id}),
        .style = styles.create(.{
            .display = .flex,
            .align_items = .center,
            .gap = .{ .rem = 0.5 },
            .padding = .{ .rem = 0.5 },
            .border_bottom_width = .{ .px = 1 },
            .border_color = ctx.theme.colors.gray[2],
        }),
    }, &.{
        h.input(ctx, Msg, .{
            .type = "checkbox",
            .checked = todo.completed,
            .onChange = .{ .toggle_todo = todo.id },
        }),
        
        h.span(ctx, Msg, .{
            .style = styles.create(.{
                .flex_grow = 1,
                .text_decoration = if (todo.completed) .line_through else .none,
                .color = if (todo.completed) 
                    ctx.theme.colors.text_secondary 
                else 
                    ctx.theme.colors.text,
            }),
        }, &.{
            h.text(ctx, Msg, todo.text),
        }),
        
        h.button(ctx, Msg, .{
            .onClick = .{ .delete_todo = todo.id },
            .style = styles.create(styles.button.danger()),
        }, &.{
            h.text(ctx, Msg, "×"),
        }),
    });
}

fn viewFooter(model: *const Model, ctx: *zui.AppContext) zui.Element(Msg) {
    const h = zui.h;
    const styles = zui.css.styles;
    
    var active_count: u32 = 0;
    for (model.todos.items) |todo| {
        if (!todo.completed) active_count += 1;
    }
    
    return h.div(ctx, Msg, .{
        .style = styles.create(.{
            .display = .flex,
            .justify_content = .space_between,
            .align_items = .center,
            .margin_top = .{ .rem = 1 },
        }),
    }, &.{
        h.span(ctx, Msg, .{}, &.{
            h.textFmt(ctx, Msg, "{d} items left", .{active_count}),
        }),
        
        h.div(ctx, Msg, .{
            .style = styles.create(.{
                .display = .flex,
                .gap = .{ .rem = 0.5 },
            }),
        }, &.{
            filterButton(.all, model.filter, ctx),
            filterButton(.active, model.filter, ctx),
            filterButton(.completed, model.filter, ctx),
        }),
        
        h.button(ctx, Msg, .{
            .onClick = .clear_completed,
            .style = styles.create(styles.button.secondary()),
        }, &.{
            h.text(ctx, Msg, "Clear completed"),
        }),
    });
}

fn filterButton(filter: Filter, current: Filter, ctx: *zui.AppContext) zui.Element(Msg) {
    const h = zui.h;
    const styles = zui.css.styles;
    const active = filter == current;
    
    return h.button(ctx, Msg, .{
        .onClick = .{ .set_filter = filter },
        .style = styles.create(if (active) 
            styles.button.primary() 
        else 
            styles.button.secondary()),
    }, &.{
        h.text(ctx, Msg, @tagName(filter)),
    });
}
```

---

## EXAMPLE 3: HTTP DATA FETCHING

### User List with API

```zig
// examples/users/src/main.zig
const std = @import("std");
const zui = @import("zui");

pub const Model = struct {
    users: RemoteData([]User),
    allocator: std.mem.Allocator,
    
    pub fn init(allocator: std.mem.Allocator) Model {
        return .{
            .users = .not_asked,
            .allocator = allocator,
        };
    }
};

pub const User = struct {
    id: u32,
    name: []const u8,
    email: []const u8,
};

pub fn RemoteData(comptime T: type) type {
    return union(enum) {
        not_asked,
        loading,
        success: T,
        failure: []const u8,
    };
}

pub const Msg = enum {
    fetch_users,
    users_loaded,
    users_failed,
};

pub fn update(model: *Model, msg: Msg, ctx: *zui.AppContext) zui.Effect(Msg) {
    _ = ctx;
    
    return switch (msg) {
        .fetch_users => {
            model.users = .loading;
            return zui.Effect.http(.{
                .method = .GET,
                .url = "https://jsonplaceholder.typicode.com/users",
                .on_success = .users_loaded,
                .on_error = .users_failed,
            });
        },
        
        .users_loaded => |data| {
            const users = parseUsers(data, model.allocator) catch {
                model.users = .{ .failure = "Parse error" };
                return zui.Effect.none;
            };
            model.users = .{ .success = users };
            return zui.Effect.none;
        },
        
        .users_failed => |error_msg| {
            model.users = .{ .failure = error_msg };
            return zui.Effect.none;
        },
    };
}

pub fn view(model: *const Model, ctx: *zui.AppContext) zui.Element(Msg) {
    const h = zui.h;
    const styles = zui.css.styles;
    
    return h.div(ctx, Msg, .{
        .style = styles.create(styles.container.centered()),
    }, &.{
        h.h1(ctx, Msg, .{}, &.{
            h.text(ctx, Msg, "Users"),
        }),
        
        viewUsersList(model, ctx),
    });
}

fn viewUsersList(model: *const Model, ctx: *zui.AppContext) zui.Element(Msg) {
    const h = zui.h;
    const styles = zui.css.styles;
    
    return switch (model.users) {
        .not_asked => h.button(ctx, Msg, .{
            .onClick = .fetch_users,
            .style = styles.create(styles.button.primary()),
        }, &.{
            h.text(ctx, Msg, "Load Users"),
        }),
        
        .loading => h.div(ctx, Msg, .{}, &.{
            h.text(ctx, Msg, "Loading..."),
        }),
        
        .success => |users| viewUsers(users, ctx),
        
        .failure => |error_msg| h.div(ctx, Msg, .{
            .style = styles.create(.{
                .color = ctx.theme.colors.danger,
            }),
        }, &.{
            h.text(ctx, Msg, error_msg),
            h.button(ctx, Msg, .{
                .onClick = .fetch_users,
                .style = styles.create(styles.button.secondary()),
            }, &.{
                h.text(ctx, Msg, "Retry"),
            }),
        }),
    };
}

fn viewUsers(users: []User, ctx: *zui.AppContext) zui.Element(Msg) {
    const h = zui.h;
    const styles = zui.css.styles;
    
    var items = ctx.allocator.alloc(zui.Element(Msg), users.len) catch &.{};
    
    for (users, 0..) |user, i| {
        items[i] = h.div(ctx, Msg, .{
            .style = styles.create(styles.card.default()),
        }, &.{
            h.h3(ctx, Msg, .{}, &.{
                h.text(ctx, Msg, user.name),
            }),
            h.p(ctx, Msg, .{}, &.{
                h.text(ctx, Msg, user.email),
            }),
        });
    }
    
    return h.div(ctx, Msg, .{
        .style = styles.create(.{
            .display = .flex,
            .flex_direction = .column,
            .gap = .{ .rem = 1 },
        }),
    }, items);
}

fn parseUsers(json: []const u8, allocator: std.mem.Allocator) ![]User {
    // In real implementation, use std.json.parseFromSlice
    // This is a simplified version
    _ = json;
    
    var users = std.ArrayList(User).init(allocator);
    try users.append(.{
        .id = 1,
        .name = "John Doe",
        .email = "john@example.com",
    });
    
    return users.toOwnedSlice();
}
```

---

## RUNNING EXAMPLES

### Build and Run

```bash
# Navigate to example
cd examples/counter

# Build
zig build

# Serve
zig build serve

# Open browser
open http://localhost:8080
```

---

## CONCLUSION

These examples demonstrate common patterns in ZUI applications: local state management, asynchronous operations, and interactive UI.

**Links:**
- [← Previous: Build System](15-build-system.md)
- [→ Next: Tests](17-tests.md)
- [↑ Back to index](../README.md)
