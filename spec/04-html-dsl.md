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

**Links:**
- [← Previous: Virtual DOM](03-virtual-dom.md)
- [→ Next: CSS Types](05-css-types.md)
- [↑ Back to index](../README.md)
