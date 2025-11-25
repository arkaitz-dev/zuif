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

## INTERNATIONALIZATION (I18N)

### I18n State in Model

```zig
// src/i18n/i18n.zig
const std = @import("std");

pub const Locale = enum {
    en,
    es,
    fr,
    de,
    ja,
    zh,

    pub fn toString(self: Locale) []const u8 {
        return switch (self) {
            .en => "en",
            .es => "es",
            .fr => "fr",
            .de => "de",
            .ja => "ja",
            .zh => "zh",
        };
    }

    pub fn fromString(str: []const u8) ?Locale {
        if (std.mem.eql(u8, str, "en")) return .en;
        if (std.mem.eql(u8, str, "es")) return .es;
        if (std.mem.eql(u8, str, "fr")) return .fr;
        if (std.mem.eql(u8, str, "de")) return .de;
        if (std.mem.eql(u8, str, "ja")) return .ja;
        if (std.mem.eql(u8, str, "zh")) return .zh;
        return null;
    }
};

pub const I18nState = struct {
    current_locale: Locale,
    fallback_locale: Locale,
    translations: std.StringHashMap(Translation),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, locale: Locale) !I18nState {
        return .{
            .current_locale = locale,
            .fallback_locale = .en,
            .translations = std.StringHashMap(Translation).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *I18nState) void {
        self.translations.deinit();
    }

    /// Get translated string
    pub fn t(self: *I18nState, key: []const u8) []const u8 {
        // Try current locale
        const locale_key = std.fmt.allocPrint(
            self.allocator,
            "{s}.{s}",
            .{ self.current_locale.toString(), key },
        ) catch return key;
        defer self.allocator.free(locale_key);

        if (self.translations.get(locale_key)) |translation| {
            return translation.value;
        }

        // Try fallback locale
        const fallback_key = std.fmt.allocPrint(
            self.allocator,
            "{s}.{s}",
            .{ self.fallback_locale.toString(), key },
        ) catch return key;
        defer self.allocator.free(fallback_key);

        if (self.translations.get(fallback_key)) |translation| {
            return translation.value;
        }

        // Return key if not found
        return key;
    }

    /// Get translated string with interpolation
    pub fn tf(
        self: *I18nState,
        key: []const u8,
        args: anytype,
    ) []const u8 {
        const template = self.t(key);
        return std.fmt.allocPrint(self.allocator, template, args) catch template;
    }

    /// Pluralization
    pub fn plural(
        self: *I18nState,
        key: []const u8,
        count: i32,
    ) []const u8 {
        const plural_key = if (count == 1)
            key
        else
            std.fmt.allocPrint(self.allocator, "{s}_plural", .{key}) catch key;

        defer if (count != 1) self.allocator.free(plural_key);

        return self.t(plural_key);
    }
};

pub const Translation = struct {
    key: []const u8,
    value: []const u8,
};
```

### Translation Loading

```zig
// Load translations from JSON
pub fn loadTranslations(
    i18n: *I18nState,
    json_data: []const u8,
) !void {
    const parsed = try std.json.parseFromSlice(
        std.json.Value,
        i18n.allocator,
        json_data,
        .{},
    );
    defer parsed.deinit();

    var it = parsed.value.object.iterator();
    while (it.next()) |entry| {
        try i18n.translations.put(entry.key_ptr.*, .{
            .key = entry.key_ptr.*,
            .value = entry.value_ptr.string,
        });
    }
}

// Example translation file: translations/en.json
// {
//   "app.title": "My Application",
//   "user.greeting": "Hello, {s}!",
//   "item.count": "{d} item",
//   "item.count_plural": "{d} items",
//   "button.save": "Save",
//   "button.cancel": "Cancel",
//   "error.network": "Network error occurred"
// }
```

### Model with I18n

```zig
pub const Model = struct {
    // ... other state ...
    i18n: I18nState,

    pub fn init(allocator: std.mem.Allocator) !Model {
        // Detect browser locale
        const locale = detectBrowserLocale() orelse .en;

        var i18n = try I18nState.init(allocator, locale);

        // Load translations for current locale
        const translations_json = try loadLocaleFile(allocator, locale);
        try loadTranslations(&i18n, translations_json);

        return .{
            // ... other initialization ...
            .i18n = i18n,
        };
    }

    pub fn deinit(self: *Model) void {
        self.i18n.deinit();
        // ... other cleanup ...
    }
};
```

### I18n Messages

```zig
pub const Msg = union(enum) {
    // ... existing messages ...

    /// Change application locale
    change_locale: Locale,

    /// Translations loaded
    translations_loaded: struct {
        locale: Locale,
        data: []const u8,
    },

    /// Translation loading failed
    translations_failed: []const u8,
};
```

### I18n Update Logic

```zig
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .change_locale => |locale| {
            model.i18n.current_locale = locale;

            // Load translations for new locale
            const locale_file = std.fmt.allocPrint(
                ctx.allocator,
                "/translations/{s}.json",
                .{locale.toString()},
            ) catch return Effect.none(Msg);

            return Effect.get(
                Msg,
                locale_file,
                |data| .{ .translations_loaded = .{
                    .locale = locale,
                    .data = data,
                }},
                |err| .{ .translations_failed = formatError(err) },
            );
        },

        .translations_loaded => |loaded| {
            loadTranslations(&model.i18n, loaded.data) catch {};
            return Effect.none(Msg);
        },

        .translations_failed => |error_msg| {
            model.error_message = error_msg;
            return Effect.none(Msg);
        },

        // ... other cases ...
    };
}
```

### Using I18n in Views

```zig
// Example: Translated UI
pub fn view(ctx: *AppContext, model: Model) Element(Msg) {
    return h.div(ctx, Msg, .{}, &.{
        // Simple translation
        h.h1(ctx, Msg, .{}, &.{
            h.text(ctx, Msg, model.i18n.t("app.title")),
        }),

        // Translation with interpolation
        h.p(ctx, Msg, .{}, &.{
            h.text(ctx, Msg, model.i18n.tf(
                "user.greeting",
                .{model.current_user.name},
            )),
        }),

        // Pluralization
        h.p(ctx, Msg, .{}, &.{
            h.text(ctx, Msg, model.i18n.tf(
                model.i18n.plural("item.count", model.items.len),
                .{model.items.len},
            )),
        }),

        // Locale selector
        localeSelector(ctx, model),
    });
}

pub fn localeSelector(ctx: *AppContext, model: Model) Element(Msg) {
    return h.select(ctx, Msg, .{
        .onChange = |e| .{ .change_locale = Locale.fromString(e.target.value) },
    }, &.{
        h.option(ctx, Msg, .{
            .value = "en",
            .selected = model.i18n.current_locale == .en,
        }, &.{h.text(ctx, Msg, "English")}),

        h.option(ctx, Msg, .{
            .value = "es",
            .selected = model.i18n.current_locale == .es,
        }, &.{h.text(ctx, Msg, "Español")}),

        h.option(ctx, Msg, .{
            .value = "fr",
            .selected = model.i18n.current_locale == .fr,
        }, &.{h.text(ctx, Msg, "Français")}),
    });
}
```

### Date/Time Formatting

```zig
// src/i18n/formatting.zig
pub const DateFormat = enum {
    short,      // 12/31/2023
    medium,     // Dec 31, 2023
    long,       // December 31, 2023
    full,       // Sunday, December 31, 2023
};

pub fn formatDate(
    allocator: std.mem.Allocator,
    timestamp: u64,
    locale: Locale,
    format: DateFormat,
) ![]const u8 {
    // Simplified formatting (real implementation would use proper locale rules)
    const date = std.time.epoch.EpochSeconds{ .secs = timestamp };

    return switch (format) {
        .short => std.fmt.allocPrint(allocator, "{d}/{d}/{d}", .{
            date.getMonth(),
            date.getDay(),
            date.getYear(),
        }),
        .medium => std.fmt.allocPrint(allocator, "{s} {d}, {d}", .{
            getMonthName(locale, date.getMonth()),
            date.getDay(),
            date.getYear(),
        }),
        // ... other formats
    };
}

pub fn formatNumber(
    allocator: std.mem.Allocator,
    number: f64,
    locale: Locale,
) ![]const u8 {
    return switch (locale) {
        .en => std.fmt.allocPrint(allocator, "{d:.2}", .{number}),
        .es, .fr, .de => std.fmt.allocPrint(allocator, "{d:.2}", .{number}), // Would use comma as decimal
        else => std.fmt.allocPrint(allocator, "{d:.2}", .{number}),
    };
}

pub fn formatCurrency(
    allocator: std.mem.Allocator,
    amount: f64,
    locale: Locale,
) ![]const u8 {
    return switch (locale) {
        .en => std.fmt.allocPrint(allocator, "${d:.2}", .{amount}),
        .es => std.fmt.allocPrint(allocator, "{d:.2}€", .{amount}),
        .fr => std.fmt.allocPrint(allocator, "{d:.2} €", .{amount}),
        .de => std.fmt.allocPrint(allocator, "{d:.2} €", .{amount}),
        .ja => std.fmt.allocPrint(allocator, "¥{d:.0}", .{amount}),
        .zh => std.fmt.allocPrint(allocator, "¥{d:.2}", .{amount}),
    };
}
```

### Complete I18n Example

```zig
// Example: Multi-language product page
pub const Model = struct {
    i18n: I18nState,
    product: Product,
    price: f64,
    stock: i32,
    release_date: u64,
};

pub const Product = struct {
    name: []const u8,
    description: []const u8,
};

pub fn view(ctx: *AppContext, model: Model) Element(Msg) {
    return h.div(ctx, Msg, .{ .class = "product-page" }, &.{
        // Product name (from database, not translated)
        h.h1(ctx, Msg, .{}, &.{h.text(ctx, Msg, model.product.name)}),

        // Stock status (translated with pluralization)
        h.p(ctx, Msg, .{ .class = "stock" }, &.{
            h.text(ctx, Msg, if (model.stock > 0)
                model.i18n.tf(
                    model.i18n.plural("product.in_stock", model.stock),
                    .{model.stock},
                )
            else
                model.i18n.t("product.out_of_stock")),
        }),

        // Price (formatted for locale)
        h.p(ctx, Msg, .{ .class = "price" }, &.{
            h.text(ctx, Msg, model.i18n.tf(
                "product.price",
                .{formatCurrency(ctx.allocator, model.price, model.i18n.current_locale)},
            )),
        }),

        // Release date (formatted for locale)
        h.p(ctx, Msg, .{ .class = "release" }, &.{
            h.text(ctx, Msg, model.i18n.tf(
                "product.released_on",
                .{formatDate(
                    ctx.allocator,
                    model.release_date,
                    model.i18n.current_locale,
                    .medium,
                )},
            )),
        }),

        // Action buttons (translated)
        h.div(ctx, Msg, .{ .class = "actions" }, &.{
            h.button(ctx, Msg, .{
                .onClick = .add_to_cart,
                .disabled = model.stock == 0,
            }, &.{
                h.text(ctx, Msg, model.i18n.t("product.add_to_cart")),
            }),

            h.button(ctx, Msg, .{
                .onClick = .add_to_wishlist,
            }, &.{
                h.text(ctx, Msg, model.i18n.t("product.add_to_wishlist")),
            }),
        }),

        // Locale switcher
        localeSelector(ctx, model),
    });
}
```

### Browser Locale Detection

```zig
// Detect browser locale via JS
extern "zui" fn js_getBrowserLocale(out_ptr: [*]u8, out_len: usize) usize;

pub fn detectBrowserLocale() ?Locale {
    var buf: [10]u8 = undefined;
    const len = js_getBrowserLocale(&buf, buf.len);

    if (len == 0) return null;

    const locale_str = buf[0..len];
    return Locale.fromString(locale_str);
}

// JavaScript implementation
// Add to zui.js:
js_getBrowserLocale: (outPtr, outLen) => {
    const locale = navigator.language || navigator.userLanguage || 'en';
    const localeShort = locale.split('-')[0]; // 'en-US' -> 'en'

    const bytes = new TextEncoder().encode(localeShort);
    const len = Math.min(bytes.length, outLen);

    new Uint8Array(memory.buffer, outPtr, len).set(bytes.subarray(0, len));

    return len;
}
```

---

## CONCLUSION

State management in ZUI follows the Elm Architecture, providing predictable unidirectional data flow with clear separation between state, actions, and update logic. Internationalization support enables building applications for global audiences with proper locale handling, translations, pluralization, and formatting.

**Links:**
- [← Previous: Effects System](11-effects-system.md)
- [→ Next: WASM Runtime](13-wasm-runtime.md)
- [↑ Back to index](../README.md)
