# 12 - STATE MANAGEMENT

**Document:** Application State Management
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

ZUI follows the Elm Architecture for state management: Model → View → Update. State is immutable by default, with explicit mutations only in the `update` function.

---

## ELM ARCHITECTURE

### Pattern Overview

```
┌──────────────────────────────────────────┐
│          ELM ARCHITECTURE                │
├──────────────────────────────────────────┤
│                                          │
│  ┌─────────┐                             │
│  │  Model  │  ←─────────┐                │
│  └────┬────┘            │                │
│       │                 │                │
│       │ view()          │ update()       │
│       ▼                 │                │
│  ┌─────────┐       ┌────┴────┐          │
│  │   View  │──────▶│   Msg   │          │
│  └─────────┘       └─────────┘          │
│                                          │
└──────────────────────────────────────────┘
```

---

## MODEL DEFINITION

### Application State

```zig
// src/main.zig
const std = @import("std");
const zui = @import("zui");

pub const Model = struct {
    // Primitive state
    count: i32,
    loading: bool,
    error_message: ?[]const u8,

    // Collections
    users: std.ArrayList(User),
    todos: std.ArrayList(Todo),

    // Nested state
    form: FormState,
    pagination: PaginationState,

    // Allocator for dynamic data
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) !Model {
        return .{
            .count = 0,
            .loading = false,
            .error_message = null,
            .users = std.ArrayList(User).init(allocator),
            .todos = std.ArrayList(Todo).init(allocator),
            .form = FormState.init(),
            .pagination = PaginationState.init(),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Model) void {
        self.users.deinit();
        self.todos.deinit();
        if (self.error_message) |msg| {
            self.allocator.free(msg);
        }
    }
};

pub const User = struct {
    id: u32,
    name: []const u8,
    email: []const u8,
    active: bool,
};

pub const Todo = struct {
    id: u32,
    title: []const u8,
    completed: bool,
    created_at: u64,
};

pub const FormState = struct {
    name: []const u8,
    email: []const u8,
    password: []const u8,
    errors: std.StringHashMap([]const u8),

    pub fn init() FormState {
        return .{
            .name = "",
            .email = "",
            .password = "",
            .errors = std.StringHashMap([]const u8).init(std.heap.page_allocator),
        };
    }

    pub fn isValid(self: FormState) bool {
        return self.errors.count() == 0;
    }
};

pub const PaginationState = struct {
    current_page: u32,
    page_size: u32,
    total_items: u32,

    pub fn init() PaginationState {
        return .{
            .current_page = 1,
            .page_size = 10,
            .total_items = 0,
        };
    }

    pub fn totalPages(self: PaginationState) u32 {
        if (self.total_items == 0) return 0;
        return (self.total_items + self.page_size - 1) / self.page_size;
    }
};
```

---

## MESSAGE DEFINITION

### Action Types

```zig
pub const Msg = union(enum) {
    // Simple actions
    increment,
    decrement,
    reset,

    // Actions with payloads
    set_count: i32,
    add_user: User,
    remove_user: u32, // id
    update_user: struct {
        id: u32,
        user: User,
    },

    // Form actions
    form_update_name: []const u8,
    form_update_email: []const u8,
    form_submit,
    form_reset,

    // Async actions
    fetch_users,
    users_loaded: []const User,
    users_failed: []const u8,

    // Pagination
    page_next,
    page_prev,
    page_goto: u32,

    // Navigation
    navigate_home,
    navigate_profile: u32, // user id

    // UI interactions
    toggle_sidebar,
    open_modal: ModalType,
    close_modal,

    pub const ModalType = enum {
        user_edit,
        user_delete,
        settings,
    };
};
```

---

## UPDATE FUNCTION

### State Transitions

```zig
pub fn update(model: *Model, msg: Msg, ctx: *zui.AppContext) zui.Effect(Msg) {
    return switch (msg) {
        // Simple state updates
        .increment => {
            model.count += 1;
            return zui.Effect.none;
        },

        .decrement => {
            model.count -= 1;
            return zui.Effect.none;
        },

        .reset => {
            model.count = 0;
            return zui.Effect.none;
        },

        .set_count => |value| {
            model.count = value;
            return zui.Effect.none;
        },

        // Collection updates
        .add_user => |user| {
            model.users.append(user) catch {};
            return zui.Effect.none;
        },

        .remove_user => |id| {
            var i: usize = 0;
            while (i < model.users.items.len) {
                if (model.users.items[i].id == id) {
                    _ = model.users.orderedRemove(i);
                    break;
                }
                i += 1;
            }
            return zui.Effect.none;
        },

        .update_user => |data| {
            for (model.users.items) |*user| {
                if (user.id == data.id) {
                    user.* = data.user;
                    break;
                }
            }
            return zui.Effect.none;
        },

        // Form updates
        .form_update_name => |name| {
            model.form.name = name;
            return zui.Effect.none;
        },

        .form_update_email => |email| {
            model.form.email = email;
            return zui.Effect.none;
        },

        .form_submit => {
            // Validate
            if (!validateForm(&model.form)) {
                return zui.Effect.none;
            }

            // Submit
            model.loading = true;
            return zui.Effect.http(.{
                .method = .POST,
                .url = "/api/users",
                .body = serializeForm(model.form),
                .on_success = .users_loaded,
                .on_error = .users_failed,
            });
        },

        .form_reset => {
            model.form = FormState.init();
            return zui.Effect.none;
        },

        // Async operations
        .fetch_users => {
            model.loading = true;
            model.error_message = null;

            return zui.Effect.http(.{
                .method = .GET,
                .url = "/api/users",
                .on_success = .users_loaded,
                .on_error = .users_failed,
            });
        },

        .users_loaded => |users| {
            model.loading = false;
            model.users.clearRetainingCapacity();
            model.users.appendSlice(users) catch {};
            return zui.Effect.none;
        },

        .users_failed => |error_msg| {
            model.loading = false;
            model.error_message = error_msg;
            return zui.Effect.none;
        },

        // Pagination
        .page_next => {
            if (model.pagination.current_page < model.pagination.totalPages()) {
                model.pagination.current_page += 1;
                return zui.Effect.batch(&.{
                    zui.Effect.timeout(100, .fetch_users),
                });
            }
            return zui.Effect.none;
        },

        .page_prev => {
            if (model.pagination.current_page > 1) {
                model.pagination.current_page -= 1;
                return zui.Effect.timeout(100, .fetch_users);
            }
            return zui.Effect.none;
        },

        .page_goto => |page| {
            if (page > 0 and page <= model.pagination.totalPages()) {
                model.pagination.current_page = page;
                return zui.Effect.timeout(100, .fetch_users);
            }
            return zui.Effect.none;
        },

        // Navigation
        .navigate_home => {
            ctx.router.navigate(routes.home, .{}) catch {};
            return zui.Effect.none;
        },

        .navigate_profile => |user_id| {
            ctx.router.navigate(routes.profile, .{ .id = user_id }) catch {};
            return zui.Effect.none;
        },

        // UI state
        .toggle_sidebar => {
            model.sidebar_open = !model.sidebar_open;
            return zui.Effect.none;
        },

        .open_modal => |modal_type| {
            model.current_modal = modal_type;
            return zui.Effect.none;
        },

        .close_modal => {
            model.current_modal = null;
            return zui.Effect.none;
        },
    };
}
```

---

## STATE PATTERNS

### Computed Properties

```zig
pub const ModelHelpers = struct {
    /// Get active users
    pub fn activeUsers(model: *const Model) []const User {
        var active = std.ArrayList(User).init(model.allocator);

        for (model.users.items) |user| {
            if (user.active) {
                active.append(user) catch {};
            }
        }

        return active.toOwnedSlice() catch &.{};
    }

    /// Get completed todos
    pub fn completedTodos(model: *const Model) []const Todo {
        var completed = std.ArrayList(Todo).init(model.allocator);

        for (model.todos.items) |todo| {
            if (todo.completed) {
                completed.append(todo) catch {};
            }
        }

        return completed.toOwnedSlice() catch &.{};
    }

    /// Calculate completion percentage
    pub fn completionRate(model: *const Model) f32 {
        if (model.todos.items.len == 0) return 0;

        var completed: u32 = 0;
        for (model.todos.items) |todo| {
            if (todo.completed) completed += 1;
        }

        return @as(f32, @floatFromInt(completed)) /
               @as(f32, @floatFromInt(model.todos.items.len));
    }
};
```

### State Validation

```zig
fn validateForm(form: *FormState) bool {
    form.errors.clearRetainingCapacity();

    // Name validation
    if (form.name.len == 0) {
        form.errors.put("name", "Name is required") catch {};
    } else if (form.name.len < 2) {
        form.errors.put("name", "Name must be at least 2 characters") catch {};
    }

    // Email validation
    if (form.email.len == 0) {
        form.errors.put("email", "Email is required") catch {};
    } else if (!isValidEmail(form.email)) {
        form.errors.put("email", "Invalid email format") catch {};
    }

    // Password validation
    if (form.password.len == 0) {
        form.errors.put("password", "Password is required") catch {};
    } else if (form.password.len < 8) {
        form.errors.put("password", "Password must be at least 8 characters") catch {};
    }

    return form.errors.count() == 0;
}

fn isValidEmail(email: []const u8) bool {
    // Simple email validation
    var has_at = false;
    var has_dot = false;

    for (email) |c| {
        if (c == '@') has_at = true;
        if (has_at and c == '.') has_dot = true;
    }

    return has_at and has_dot;
}
```

---

## STATE PERSISTENCE

### LocalStorage Integration

```zig
pub const Persistence = struct {
    /// Save model to localStorage
    pub fn save(model: *const Model) zui.Effect(Msg) {
        const json = serializeModel(model) catch return zui.Effect.none;

        return zui.Effect.localStorage(.{
            .set = .{
                .key = "app_state",
                .value = json,
                .on_complete = .state_saved,
            },
        });
    }

    /// Load model from localStorage
    pub fn load() zui.Effect(Msg) {
        return zui.Effect.localStorage(.{
            .get = .{
                .key = "app_state",
                .on_success = .state_loaded,
            },
        });
    }

    fn serializeModel(model: *const Model) ![]const u8 {
        var json = std.ArrayList(u8).init(model.allocator);
        var writer = json.writer();

        try writer.writeAll("{");
        try writer.print("\"count\":{d},", .{model.count});
        try writer.print("\"loading\":{},", .{model.loading});
        // ... serialize other fields
        try writer.writeAll("}");

        return json.toOwnedSlice();
    }
};
```

---

## UNDO/REDO PATTERN

```zig
pub const History = struct {
    past: std.ArrayList(Model),
    present: Model,
    future: std.ArrayList(Model),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, initial: Model) !History {
        return .{
            .past = std.ArrayList(Model).init(allocator),
            .present = initial,
            .future = std.ArrayList(Model).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *History) void {
        self.past.deinit();
        self.future.deinit();
    }

    pub fn push(self: *History, new_state: Model) !void {
        try self.past.append(self.present);
        self.present = new_state;
        self.future.clearRetainingCapacity();
    }

    pub fn undo(self: *History) ?Model {
        if (self.past.items.len == 0) return null;

        const previous = self.past.pop();
        self.future.append(self.present) catch {};
        self.present = previous;

        return self.present;
    }

    pub fn redo(self: *History) ?Model {
        if (self.future.items.len == 0) return null;

        const next = self.future.pop();
        self.past.append(self.present) catch {};
        self.present = next;

        return self.present;
    }

    pub fn canUndo(self: *const History) bool {
        return self.past.items.len > 0;
    }

    pub fn canRedo(self: *const History) bool {
        return self.future.items.len > 0;
    }
};
```

---

## BEST PRACTICES

### 1. Keep Model Flat
```zig
// ✓ GOOD: Flat structure
pub const Model = struct {
    user_id: u32,
    user_name: []const u8,
    user_email: []const u8,
};

// ✗ AVOID: Deep nesting makes updates complex
pub const Model = struct {
    user: struct {
        profile: struct {
            personal: struct {
                name: []const u8,
            },
        },
    },
};
```

### 2. Use Tagged Unions for Variants
```zig
pub const RemoteData = union(enum) {
    not_asked,
    loading,
    success: []const User,
    failure: []const u8,
};
```

### 3. Separate UI State from Data
```zig
pub const Model = struct {
    // Data
    users: []const User,

    // UI State
    selected_user_id: ?u32,
    sidebar_open: bool,
    current_modal: ?ModalType,
};
```

---

## CONCLUSION

State management in ZUI follows the Elm Architecture, providing predictable unidirectional data flow with clear separation between state, actions, and update logic.

**Links:**
- [← Previous: Effects System](11-effects-system.md)
- [→ Next: WASM Runtime](13-wasm-runtime.md)
- [↑ Back to index](../README.md)
