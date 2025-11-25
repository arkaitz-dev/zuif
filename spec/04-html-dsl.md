# 04 - HTML DSL

**Document:** Type-Safe HTML DSL
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

ZUI's HTML DSL provides type-safe functions for building HTML elements. All functions require an explicit context and return elements parameterized by the application's message type.

---

## ELEMENT CONSTRUCTORS

### Common Elements

```zig
// src/html/html.zig
const std = @import("std");
const Element = @import("../vdom/element.zig").Element;
const Attrs = @import("attrs.zig").Attrs;
const AppContext = @import("../core/context.zig").AppContext;

/// HTML namespace
pub const h = struct {
    // Container elements
    pub fn div(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("div", attrs.key, attrs, children);
    }

    pub fn span(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("span", attrs.key, attrs, children);
    }

    pub fn section(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("section", attrs.key, attrs, children);
    }

    pub fn article(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("article", attrs.key, attrs, children);
    }

    pub fn header(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("header", attrs.key, attrs, children);
    }

    pub fn footer(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("footer", attrs.key, attrs, children);
    }

    pub fn main(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("main", attrs.key, attrs, children);
    }

    pub fn aside(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("aside", attrs.key, attrs, children);
    }

    pub fn nav(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("nav", attrs.key, attrs, children);
    }

    // Text elements
    pub fn h1(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("h1", attrs.key, attrs, children);
    }

    pub fn h2(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("h2", attrs.key, attrs, children);
    }

    pub fn h3(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("h3", attrs.key, attrs, children);
    }

    pub fn h4(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("h4", attrs.key, attrs, children);
    }

    pub fn h5(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("h5", attrs.key, attrs, children);
    }

    pub fn h6(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("h6", attrs.key, attrs, children);
    }

    pub fn p(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("p", attrs.key, attrs, children);
    }

    pub fn a(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("a", attrs.key, attrs, children);
    }

    pub fn strong(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("strong", attrs.key, attrs, children);
    }

    pub fn em(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("em", attrs.key, attrs, children);
    }

    pub fn code(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("code", attrs.key, attrs, children);
    }

    pub fn pre(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("pre", attrs.key, attrs, children);
    }

    // List elements
    pub fn ul(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("ul", attrs.key, attrs, children);
    }

    pub fn ol(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("ol", attrs.key, attrs, children);
    }

    pub fn li(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("li", attrs.key, attrs, children);
    }

    // Form elements
    pub fn form(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("form", attrs.key, attrs, children);
    }

    pub fn input(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg)) Element(Msg) {
        return Element(Msg).element("input", attrs.key, attrs, &.{});
    }

    pub fn textarea(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("textarea", attrs.key, attrs, children);
    }

    pub fn button(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("button", attrs.key, attrs, children);
    }

    pub fn select(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("select", attrs.key, attrs, children);
    }

    pub fn option(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("option", attrs.key, attrs, children);
    }

    pub fn label(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("label", attrs.key, attrs, children);
    }

    // Media elements
    pub fn img(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg)) Element(Msg) {
        return Element(Msg).element("img", attrs.key, attrs, &.{});
    }

    pub fn video(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("video", attrs.key, attrs, children);
    }

    pub fn audio(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("audio", attrs.key, attrs, children);
    }

    // Table elements
    pub fn table(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("table", attrs.key, attrs, children);
    }

    pub fn thead(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("thead", attrs.key, attrs, children);
    }

    pub fn tbody(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("tbody", attrs.key, attrs, children);
    }

    pub fn tr(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("tr", attrs.key, attrs, children);
    }

    pub fn th(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("th", attrs.key, attrs, children);
    }

    pub fn td(ctx: *AppContext, comptime Msg: type, attrs: Attrs(Msg), children: []Element(Msg)) Element(Msg) {
        return Element(Msg).element("td", attrs.key, attrs, children);
    }

    // Text node
    pub fn text(ctx: *AppContext, comptime Msg: type, content: []const u8) Element(Msg) {
        _ = ctx;
        return Element(Msg).text(content);
    }

    pub fn textFmt(ctx: *AppContext, comptime Msg: type, comptime fmt: []const u8, args: anytype) Element(Msg) {
        const content = std.fmt.allocPrint(ctx.allocator, fmt, args) catch return Element(Msg).none;
        return Element(Msg).text(content);
    }

    // Empty node
    pub fn none(ctx: *AppContext, comptime Msg: type) Element(Msg) {
        _ = ctx;
        return Element(Msg).none;
    }
};
```

---

## ATTRIBUTES

```zig
// src/html/attrs.zig
const std = @import("std");

pub fn Attrs(comptime Msg: type) type {
    return struct {
        map: std.StringHashMap(AttrValue),
        key: ?[]const u8,

        const AttrValue = union(enum) {
            string: []const u8,
            bool: bool,
            event: Msg,
        };

        pub fn init(allocator: std.mem.Allocator) @This() {
            return .{
                .map = std.StringHashMap(AttrValue).init(allocator),
                .key = null,
            };
        }

        pub fn deinit(self: *@This()) void {
            self.map.deinit();
        }

        // Common attributes
        pub fn id(self: *@This(), value: []const u8) !void {
            try self.map.put("id", .{ .string = value });
        }

        pub fn class(self: *@This(), value: []const u8) !void {
            try self.map.put("class", .{ .string = value });
        }

        pub fn style(self: *@This(), value: []const u8) !void {
            try self.map.put("style", .{ .string = value });
        }

        pub fn href(self: *@This(), value: []const u8) !void {
            try self.map.put("href", .{ .string = value });
        }

        pub fn src(self: *@This(), value: []const u8) !void {
            try self.map.put("src", .{ .string = value });
        }

        pub fn alt(self: *@This(), value: []const u8) !void {
            try self.map.put("alt", .{ .string = value });
        }

        pub fn title(self: *@This(), value: []const u8) !void {
            try self.map.put("title", .{ .string = value });
        }

        pub fn placeholder(self: *@This(), value: []const u8) !void {
            try self.map.put("placeholder", .{ .string = value });
        }

        pub fn value(self: *@This(), val: []const u8) !void {
            try self.map.put("value", .{ .string = val });
        }

        pub fn type(self: *@This(), value: []const u8) !void {
            try self.map.put("type", .{ .string = value });
        }

        pub fn name(self: *@This(), value: []const u8) !void {
            try self.map.put("name", .{ .string = value });
        }

        // Boolean attributes
        pub fn disabled(self: *@This(), value: bool) !void {
            try self.map.put("disabled", .{ .bool = value });
        }

        pub fn checked(self: *@This(), value: bool) !void {
            try self.map.put("checked", .{ .bool = value });
        }

        pub fn readonly(self: *@This(), value: bool) !void {
            try self.map.put("readonly", .{ .bool = value });
        }

        pub fn required(self: *@This(), value: bool) !void {
            try self.map.put("required", .{ .bool = value });
        }

        // Events
        pub fn onClick(self: *@This(), msg: Msg) !void {
            try self.map.put("onclick", .{ .event = msg });
        }

        pub fn onInput(self: *@This(), msg: Msg) !void {
            try self.map.put("oninput", .{ .event = msg });
        }

        pub fn onChange(self: *@This(), msg: Msg) !void {
            try self.map.put("onchange", .{ .event = msg });
        }

        pub fn onSubmit(self: *@This(), msg: Msg) !void {
            try self.map.put("onsubmit", .{ .event = msg });
        }

        pub fn onKeyDown(self: *@This(), msg: Msg) !void {
            try self.map.put("onkeydown", .{ .event = msg });
        }

        pub fn onKeyUp(self: *@This(), msg: Msg) !void {
            try self.map.put("onkeyup", .{ .event = msg });
        }

        pub fn onFocus(self: *@This(), msg: Msg) !void {
            try self.map.put("onfocus", .{ .event = msg });
        }

        pub fn onBlur(self: *@This(), msg: Msg) !void {
            try self.map.put("onblur", .{ .event = msg });
        }

        // Key for reconciliation
        pub fn setKey(self: *@This(), k: []const u8) void {
            self.key = k;
        }

        // Iterator
        pub const Iterator = struct {
            inner: std.StringHashMap(AttrValue).Iterator,

            pub fn next(self: *Iterator) ?struct {
                key: []const u8,
                value: []const u8,
            } {
                const entry = self.inner.next() orelse return null;

                const value_str = switch (entry.value_ptr.*) {
                    .string => |s| s,
                    .bool => |b| if (b) "true" else "false",
                    .event => "event-handler",
                };

                return .{
                    .key = entry.key_ptr.*,
                    .value = value_str,
                };
            }
        };

        pub fn iterator(self: *const @This()) Iterator {
            return .{ .inner = self.map.iterator() };
        }
    };
}
```

---

## USAGE EXAMPLES

```zig
// Simple element
h.div(ctx, Msg, .{}, &.{
    h.text(ctx, Msg, "Hello World"),
});

// With attributes
h.button(ctx, Msg, .{
    .class = "btn btn-primary",
    .onClick = .button_clicked,
}, &.{
    h.text(ctx, Msg, "Click me"),
});

// Nested structure
h.div(ctx, Msg, .{
    .class = "container",
}, &.{
    h.h1(ctx, Msg, .{}, &.{
        h.text(ctx, Msg, "Title"),
    }),
    h.p(ctx, Msg, .{}, &.{
        h.text(ctx, Msg, "Paragraph"),
    }),
});

// Forms
h.form(ctx, Msg, .{
    .onSubmit = .form_submitted,
}, &.{
    h.input(ctx, Msg, .{
        .type = "text",
        .placeholder = "Enter name",
        .value = model.name,
        .onInput = .name_changed,
    }),
    h.button(ctx, Msg, .{
        .type = "submit",
    }, &.{
        h.text(ctx, Msg, "Submit"),
    }),
});
```

---

## FORMS & VALIDATION

### Form Field Helpers

Enhanced form field constructors with validation support:

```zig
// src/html/forms.zig
const std = @import("std");
const Element = @import("../vdom/element.zig").Element;
const Attrs = @import("attrs.zig").Attrs;

pub const forms = struct {
    /// Text input with validation
    pub fn textField(
        ctx: *AppContext,
        comptime Msg: type,
        opts: struct {
            value: []const u8,
            placeholder: ?[]const u8 = null,
            disabled: bool = false,
            onInput: Msg,
            errors: []const []const u8 = &.{},
        },
    ) Element(Msg) {
        var attrs = Attrs(Msg).init(ctx.allocator);
        attrs.type("text") catch {};
        attrs.value(opts.value) catch {};
        if (opts.placeholder) |p| attrs.placeholder(p) catch {};
        attrs.disabled(opts.disabled) catch {};
        attrs.onInput(opts.onInput) catch {};
        if (opts.errors.len > 0) {
            attrs.class("error") catch {};
            attrs.ariaInvalid(true) catch {};
        }

        return Element(Msg).element("input", null, attrs, &.{});
    }

    /// Email input with built-in validation
    pub fn emailField(
        ctx: *AppContext,
        comptime Msg: type,
        opts: struct {
            value: []const u8,
            placeholder: ?[]const u8 = null,
            required: bool = false,
            onInput: Msg,
            errors: []const []const u8 = &.{},
        },
    ) Element(Msg) {
        var attrs = Attrs(Msg).init(ctx.allocator);
        attrs.type("email") catch {};
        attrs.value(opts.value) catch {};
        if (opts.placeholder) |p| attrs.placeholder(p) catch {};
        attrs.required(opts.required) catch {};
        attrs.onInput(opts.onInput) catch {};
        if (opts.errors.len > 0) {
            attrs.ariaInvalid(true) catch {};
        }

        return Element(Msg).element("input", null, attrs, &.{});
    }

    /// Password input
    pub fn passwordField(
        ctx: *AppContext,
        comptime Msg: type,
        opts: struct {
            value: []const u8,
            placeholder: ?[]const u8 = null,
            required: bool = false,
            onInput: Msg,
        },
    ) Element(Msg) {
        var attrs = Attrs(Msg).init(ctx.allocator);
        attrs.type("password") catch {};
        attrs.value(opts.value) catch {};
        if (opts.placeholder) |p| attrs.placeholder(p) catch {};
        attrs.required(opts.required) catch {};
        attrs.onInput(opts.onInput) catch {};

        return Element(Msg).element("input", null, attrs, &.{});
    }

    /// Number input with min/max
    pub fn numberField(
        ctx: *AppContext,
        comptime Msg: type,
        opts: struct {
            value: []const u8,
            min: ?i32 = null,
            max: ?i32 = null,
            step: ?i32 = null,
            onInput: Msg,
        },
    ) Element(Msg) {
        var attrs = Attrs(Msg).init(ctx.allocator);
        attrs.type("number") catch {};
        attrs.value(opts.value) catch {};
        if (opts.min) |m| {
            const min_str = std.fmt.allocPrint(ctx.allocator, "{d}", .{m}) catch "";
            attrs.min(min_str) catch {};
        }
        if (opts.max) |m| {
            const max_str = std.fmt.allocPrint(ctx.allocator, "{d}", .{m}) catch "";
            attrs.max(max_str) catch {};
        }
        attrs.onInput(opts.onInput) catch {};

        return Element(Msg).element("input", null, attrs, &.{});
    }

    /// Checkbox with label
    pub fn checkboxField(
        ctx: *AppContext,
        comptime Msg: type,
        opts: struct {
            checked: bool,
            label: []const u8,
            onChange: Msg,
        },
    ) Element(Msg) {
        return h.label(ctx, Msg, .{}, &.{
            h.input(ctx, Msg, .{
                .type = "checkbox",
                .checked = opts.checked,
                .onChange = opts.onChange,
            }),
            h.text(ctx, Msg, " "),
            h.text(ctx, Msg, opts.label),
        });
    }

    /// Display field errors
    pub fn fieldErrors(
        ctx: *AppContext,
        comptime Msg: type,
        errors: []const []const u8,
    ) Element(Msg) {
        if (errors.len == 0) return h.none(ctx, Msg);

        const error_elements = ctx.allocator.alloc(Element(Msg), errors.len) catch return h.none(ctx, Msg);
        for (errors, 0..) |err, i| {
            error_elements[i] = h.div(ctx, Msg, .{ .class = "error-message" }, &.{
                h.text(ctx, Msg, err),
            });
        }

        return h.div(ctx, Msg, .{ .class = "field-errors", .role = "alert" }, error_elements);
    }
};
```

### Validation Helpers

```zig
// src/html/validation.zig
const std = @import("std");

pub const Validator = struct {
    /// Validates required field
    pub fn required(value: []const u8, message: []const u8) ?[]const u8 {
        if (value.len == 0) return message;
        return null;
    }

    /// Validates email format (basic check)
    pub fn email(value: []const u8, message: []const u8) ?[]const u8 {
        if (value.len == 0) return null;
        const has_at = std.mem.indexOf(u8, value, "@") != null;
        const has_dot = std.mem.indexOf(u8, value, ".") != null;
        if (!has_at or !has_dot) return message;
        return null;
    }

    /// Validates minimum length
    pub fn minLength(value: []const u8, min: usize, message: []const u8) ?[]const u8 {
        if (value.len < min) return message;
        return null;
    }

    /// Validates maximum length
    pub fn maxLength(value: []const u8, max: usize, message: []const u8) ?[]const u8 {
        if (value.len > max) return message;
        return null;
    }

    /// Validates number range
    pub fn numberRange(value: i32, min: ?i32, max: ?i32, message: []const u8) ?[]const u8 {
        if (min) |m| {
            if (value < m) return message;
        }
        if (max) |m| {
            if (value > m) return message;
        }
        return null;
    }

    /// Pattern matching (simplified)
    pub fn pattern(value: []const u8, comptime pat: []const u8, message: []const u8) ?[]const u8 {
        // Basic pattern matching - can be extended with regex support
        _ = pat;
        if (value.len == 0) return message;
        return null;
    }
};
```

### Form Example

```zig
// Example: Login form with validation
pub fn viewLoginForm(ctx: *AppContext, model: Model) Element(Msg) {
    return forms.div(ctx, Msg, .{ .class = "login-form" }, &.{
        h.h2(ctx, Msg, .{}, &.{h.text(ctx, Msg, "Login")}),

        h.form(ctx, Msg, .{ .onSubmit = .login_submitted }, &.{
            // Email field
            h.div(ctx, Msg, .{ .class = "form-group" }, &.{
                h.label(ctx, Msg, .{}, &.{h.text(ctx, Msg, "Email")}),
                forms.emailField(ctx, Msg, .{
                    .value = model.email,
                    .placeholder = "you@example.com",
                    .required = true,
                    .onInput = .email_changed,
                    .errors = model.email_errors,
                }),
                forms.fieldErrors(ctx, Msg, model.email_errors),
            }),

            // Password field
            h.div(ctx, Msg, .{ .class = "form-group" }, &.{
                h.label(ctx, Msg, .{}, &.{h.text(ctx, Msg, "Password")}),
                forms.passwordField(ctx, Msg, .{
                    .value = model.password,
                    .required = true,
                    .onInput = .password_changed,
                }),
            }),

            // Remember me checkbox
            h.div(ctx, Msg, .{ .class = "form-group" }, &.{
                forms.checkboxField(ctx, Msg, .{
                    .checked = model.remember_me,
                    .label = "Remember me",
                    .onChange = .remember_me_toggled,
                }),
            }),

            // Submit button
            h.button(ctx, Msg, .{
                .type = "submit",
                .disabled = model.is_submitting,
            }, &.{
                h.text(ctx, Msg, if (model.is_submitting) "Logging in..." else "Login"),
            }),
        }),
    });
}

// Validation in update function
pub fn validateEmail(email: []const u8, allocator: std.mem.Allocator) ![][]const u8 {
    var errors = std.ArrayList([]const u8).init(allocator);

    if (Validator.required(email, "Email is required")) |err| {
        try errors.append(err);
    }
    if (Validator.email(email, "Invalid email format")) |err| {
        try errors.append(err);
    }

    return errors.toOwnedSlice();
}
```

---

## ACCESSIBILITY (ARIA)

### ARIA Attributes

Extended attributes for accessibility:

```zig
// Additional ARIA methods in src/html/attrs.zig
pub fn Attrs(comptime Msg: type) type {
    return struct {
        // ... existing fields ...

        // ARIA attributes
        pub fn role(self: *@This(), value: []const u8) !void {
            try self.map.put("role", .{ .string = value });
        }

        pub fn ariaLabel(self: *@This(), value: []const u8) !void {
            try self.map.put("aria-label", .{ .string = value });
        }

        pub fn ariaLabelledBy(self: *@This(), value: []const u8) !void {
            try self.map.put("aria-labelledby", .{ .string = value });
        }

        pub fn ariaDescribedBy(self: *@This(), value: []const u8) !void {
            try self.map.put("aria-describedby", .{ .string = value });
        }

        pub fn ariaHidden(self: *@This(), value: bool) !void {
            try self.map.put("aria-hidden", .{ .bool = value });
        }

        pub fn ariaExpanded(self: *@This(), value: bool) !void {
            try self.map.put("aria-expanded", .{ .bool = value });
        }

        pub fn ariaPressed(self: *@This(), value: bool) !void {
            try self.map.put("aria-pressed", .{ .bool = value });
        }

        pub fn ariaChecked(self: *@This(), value: bool) !void {
            try self.map.put("aria-checked", .{ .bool = value });
        }

        pub fn ariaSelected(self: *@This(), value: bool) !void {
            try self.map.put("aria-selected", .{ .bool = value });
        }

        pub fn ariaInvalid(self: *@This(), value: bool) !void {
            try self.map.put("aria-invalid", .{ .bool = value });
        }

        pub fn ariaLive(self: *@This(), value: []const u8) !void {
            // value should be: "polite", "assertive", or "off"
            try self.map.put("aria-live", .{ .string = value });
        }

        pub fn ariaAtomic(self: *@This(), value: bool) !void {
            try self.map.put("aria-atomic", .{ .bool = value });
        }

        pub fn tabIndex(self: *@This(), value: i32) !void {
            const val_str = std.fmt.allocPrint(self.map.allocator, "{d}", .{value}) catch return;
            try self.map.put("tabindex", .{ .string = val_str });
        }
    };
}
```

### Accessibility Patterns

```zig
// src/html/a11y.zig
const std = @import("std");
const Element = @import("../vdom/element.zig").Element;

pub const a11y = struct {
    /// Accessible button
    pub fn button(
        ctx: *AppContext,
        comptime Msg: type,
        opts: struct {
            label: []const u8,
            onClick: Msg,
            disabled: bool = false,
            ariaPressed: ?bool = null,
        },
        children: []Element(Msg),
    ) Element(Msg) {
        var attrs = Attrs(Msg).init(ctx.allocator);
        attrs.onClick(opts.onClick) catch {};
        attrs.disabled(opts.disabled) catch {};
        attrs.ariaLabel(opts.label) catch {};
        if (opts.ariaPressed) |pressed| {
            attrs.ariaPressed(pressed) catch {};
        }

        return h.button(ctx, Msg, attrs, children);
    }

    /// Live region for announcements
    pub fn liveRegion(
        ctx: *AppContext,
        comptime Msg: type,
        message: []const u8,
        politeness: enum { polite, assertive },
    ) Element(Msg) {
        const politeness_str = if (politeness == .polite) "polite" else "assertive";

        return h.div(ctx, Msg, .{
            .role = "status",
            .ariaLive = politeness_str,
            .ariaAtomic = true,
        }, &.{
            h.text(ctx, Msg, message),
        });
    }

    /// Skip link for keyboard navigation
    pub fn skipLink(
        ctx: *AppContext,
        comptime Msg: type,
        target: []const u8,
        label: []const u8,
    ) Element(Msg) {
        return h.a(ctx, Msg, .{
            .href = target,
            .class = "skip-link",
        }, &.{
            h.text(ctx, Msg, label),
        });
    }

    /// Accessible modal/dialog
    pub fn dialog(
        ctx: *AppContext,
        comptime Msg: type,
        opts: struct {
            title: []const u8,
            isOpen: bool,
            onClose: Msg,
        },
        children: []Element(Msg),
    ) Element(Msg) {
        if (!opts.isOpen) return h.none(ctx, Msg);

        return h.div(ctx, Msg, .{
            .role = "dialog",
            .ariaModal = true,
            .ariaLabelledBy = "dialog-title",
        }, &.{
            h.div(ctx, Msg, .{ .class = "dialog-header" }, &.{
                h.h2(ctx, Msg, .{ .id = "dialog-title" }, &.{
                    h.text(ctx, Msg, opts.title),
                }),
                h.button(ctx, Msg, .{
                    .onClick = opts.onClose,
                    .ariaLabel = "Close dialog",
                }, &.{
                    h.text(ctx, Msg, "×"),
                }),
            }),
            h.div(ctx, Msg, .{ .class = "dialog-content" }, children),
        });
    }
};
```

### Accessibility Example

```zig
// Example: Accessible navigation
pub fn viewNav(ctx: *AppContext, model: Model) Element(Msg) {
    return h.nav(ctx, Msg, .{ .ariaLabel = "Main navigation" }, &.{
        // Skip link for keyboard users
        a11y.skipLink(ctx, Msg, "#main-content", "Skip to main content"),

        h.ul(ctx, Msg, .{}, &.{
            h.li(ctx, Msg, .{}, &.{
                h.a(ctx, Msg, .{
                    .href = "/",
                    .ariaCurrentPage = if (model.currentPage == .home) true else false,
                }, &.{
                    h.text(ctx, Msg, "Home"),
                }),
            }),
            h.li(ctx, Msg, .{}, &.{
                h.a(ctx, Msg, .{
                    .href = "/about",
                    .ariaCurrentPage = if (model.currentPage == .about) true else false,
                }, &.{
                    h.text(ctx, Msg, "About"),
                }),
            }),
        }),
    });
}

// Example: Accessible form with live validation feedback
pub fn viewSignupForm(ctx: *AppContext, model: Model) Element(Msg) {
    return h.div(ctx, Msg, .{}, &.{
        h.form(ctx, Msg, .{ .onSubmit = .signup_submitted }, &.{
            // Username field with ARIA
            h.div(ctx, Msg, .{ .class = "form-group" }, &.{
                h.label(ctx, Msg, .{ .for = "username" }, &.{
                    h.text(ctx, Msg, "Username"),
                }),
                h.input(ctx, Msg, .{
                    .id = "username",
                    .type = "text",
                    .value = model.username,
                    .onInput = .username_changed,
                    .ariaInvalid = model.username_errors.len > 0,
                    .ariaDescribedBy = if (model.username_errors.len > 0) "username-error" else null,
                }),
                if (model.username_errors.len > 0)
                    h.div(ctx, Msg, .{
                        .id = "username-error",
                        .role = "alert",
                    }, &.{
                        h.text(ctx, Msg, model.username_errors[0]),
                    })
                else
                    h.none(ctx, Msg),
            }),

            h.button(ctx, Msg, .{ .type = "submit" }, &.{
                h.text(ctx, Msg, "Sign Up"),
            }),
        }),

        // Live region for status messages
        if (model.statusMessage.len > 0)
            a11y.liveRegion(ctx, Msg, model.statusMessage, .polite)
        else
            h.none(ctx, Msg),
    });
}
```

**Links:**
- [← Previous: Virtual DOM](03-virtual-dom.md)
- [→ Next: CSS Types](05-css-types.md)
- [↑ Back to index](../README.md)
