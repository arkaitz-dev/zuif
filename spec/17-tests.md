# 17 - TESTS

**Document:** Testing Suite
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

ZUI includes a complete test suite: unit tests for individual functions, integration tests for complete components, and benchmarks for performance.

---

## TEST STRUCTURE

### Directory Layout

```
tests/
├── unit/
│   ├── vdom_test.zig
│   ├── diff_test.zig
│   ├── css_test.zig
│   ├── router_test.zig
│   └── effects_test.zig
├── integration/
│   ├── counter_test.zig
│   ├── todo_test.zig
│   └── http_test.zig
└── bench/
    ├── vdom_bench.zig
    ├── css_bench.zig
    └── router_bench.zig
```

---

## UNIT TESTS

### Virtual DOM Tests

```zig
// tests/unit/vdom_test.zig
const std = @import("std");
const testing = std.testing;
const Element = @import("zui").vdom.Element;

test "Element creation" {
    const element = Element(u32).element("div", null, .{}, &.{});
    try testing.expect(element == .element);
    try testing.expectEqualStrings("div", element.element.tag);
}

test "Text node creation" {
    const text = Element(u32).text("Hello");
    try testing.expect(text == .text);
    try testing.expectEqualStrings("Hello", text.text);
}

test "Element with children" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    var children = std.ArrayList(Element(u32)).init(allocator);
    try children.append(Element(u32).text("Child 1"));
    try children.append(Element(u32).text("Child 2"));
    
    const element = Element(u32).element(
        "div",
        null,
        .{},
        try children.toOwnedSlice(),
    );
    
    try testing.expectEqual(@as(usize, 2), element.element.children.len);
}
```

### Diff Algorithm Tests

```zig
// tests/unit/diff_test.zig
const std = @import("std");
const testing = std.testing;
const diff = @import("zui").vdom.diff;
const Element = @import("zui").vdom.Element;

test "Diff identical elements" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const old = Element(u32).text("Hello");
    const new = Element(u32).text("Hello");
    
    const patches = try diff(u32, allocator, old, new, 0);
    defer allocator.free(patches);
    
    try testing.expectEqual(@as(usize, 0), patches.len);
}

test "Diff different text" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const old = Element(u32).text("Hello");
    const new = Element(u32).text("World");
    
    const patches = try diff(u32, allocator, old, new, 0);
    defer allocator.free(patches);
    
    try testing.expectEqual(@as(usize, 1), patches.len);
    try testing.expect(patches[0] == .update_text);
}

test "Diff element to text" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const old = Element(u32).element("div", null, .{}, &.{});
    const new = Element(u32).text("Text");
    
    const patches = try diff(u32, allocator, old, new, 0);
    defer allocator.free(patches);
    
    try testing.expectEqual(@as(usize, 1), patches.len);
    try testing.expect(patches[0] == .replace);
}

test "Diff keyed children reordering" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    var old_children = std.ArrayList(Element(u32)).init(allocator);
    try old_children.append(Element(u32).element("div", "a", .{}, &.{}));
    try old_children.append(Element(u32).element("div", "b", .{}, &.{}));
    
    var new_children = std.ArrayList(Element(u32)).init(allocator);
    try new_children.append(Element(u32).element("div", "b", .{}, &.{}));
    try new_children.append(Element(u32).element("div", "a", .{}, &.{}));
    
    const old = Element(u32).element("ul", null, .{}, try old_children.toOwnedSlice());
    const new = Element(u32).element("ul", null, .{}, try new_children.toOwnedSlice());
    
    const patches = try diff(u32, allocator, old, new, 0);
    defer allocator.free(patches);
    
    // Should detect reordering
    var has_reorder = false;
    for (patches) |patch| {
        if (patch == .reorder) has_reorder = true;
    }
    try testing.expect(has_reorder);
}
```

### CSS Tests

```zig
// tests/unit/css_test.zig
const std = @import("std");
const testing = std.testing;
const StyleRule = @import("zui").css.StyleRule;
const types = @import("zui").css.types;

test "StyleRule to CSS" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const rule = StyleRule{
        .display = .flex,
        .flex_direction = .column,
        .gap = .{ .rem = 1 },
    };
    
    const css = try rule.toCss(allocator);
    defer allocator.free(css);
    
    try testing.expect(std.mem.indexOf(u8, css, "display: flex") != null);
    try testing.expect(std.mem.indexOf(u8, css, "flex-direction: column") != null);
    try testing.expect(std.mem.indexOf(u8, css, "gap: 1rem") != null);
}

test "StyleRule merge" {
    const base = StyleRule{
        .display = .flex,
        .padding = .{ .rem = 1 },
    };
    
    const override = StyleRule{
        .padding = .{ .rem = 2 },
        .gap = .{ .rem = 0.5 },
    };
    
    const merged = base.merge(override);
    
    try testing.expectEqual(types.Display.flex, merged.display.?);
    try testing.expectEqual(@as(f32, 2), merged.padding.?.rem);
    try testing.expectEqual(@as(f32, 0.5), merged.gap.?.rem);
}

test "Color to CSS" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const hex = types.Color{ .hex = 0xff0000 };
    const css = try hex.toCss(allocator);
    defer allocator.free(css);
    
    try testing.expectEqualStrings("#ff0000", css);
}
```

### Router Tests

```zig
// tests/unit/router_test.zig
const std = @import("std");
const testing = std.testing;
const Router = @import("zui").router.Router;
const Route = @import("zui").router.Route;

test "Static route" {
    const route = Route.static("/about");
    try testing.expectEqualStrings("/about", route.path);
    try testing.expect(route.params_type == null);
}

test "Parameterized route" {
    const ParamsType = struct { id: u32 };
    const route = Route.param("/user/:id", ParamsType);
    
    try testing.expectEqualStrings("/user/:id", route.path);
    try testing.expect(route.params_type != null);
}

test "Route build with params" {
    var arena = std.heap.ArenaAllocator.init(testing.allocator);
    defer arena.deinit();
    const allocator = arena.allocator();
    
    const ParamsType = struct { id: u32 };
    const route = Route.param("/user/:id", ParamsType);
    
    const path = try route.build(allocator, .{ .id = 42 });
    defer allocator.free(path);
    
    try testing.expectEqualStrings("/user/42", path);
}

test "Router navigation" {
    var router = try Router.init(testing.allocator);
    defer router.deinit();
    
    try router.push("/home");
    try testing.expectEqualStrings("/home", router.currentPath());
    
    try router.push("/about");
    try testing.expectEqualStrings("/about", router.currentPath());
}
```

---

## INTEGRATION TESTS

### Counter App Test

```zig
// tests/integration/counter_test.zig
const std = @import("std");
const testing = std.testing;
const counter = @import("../../examples/counter/src/main.zig");
const TestContext = @import("zui").TestContext;

test "Counter initialization" {
    const model = counter.init();
    try testing.expectEqual(@as(i32, 0), model.count);
}

test "Counter increment" {
    var ctx = try TestContext.init();
    defer ctx.deinit();
    
    var model = counter.init();
    _ = counter.update(&model, .increment, &ctx.ctx);
    
    try testing.expectEqual(@as(i32, 1), model.count);
}

test "Counter decrement" {
    var ctx = try TestContext.init();
    defer ctx.deinit();
    
    var model = counter.init();
    _ = counter.update(&model, .decrement, &ctx.ctx);
    
    try testing.expectEqual(@as(i32, -1), model.count);
}

test "Counter reset" {
    var ctx = try TestContext.init();
    defer ctx.deinit();
    
    var model = counter.init();
    _ = counter.update(&model, .increment, &ctx.ctx);
    _ = counter.update(&model, .increment, &ctx.ctx);
    try testing.expectEqual(@as(i32, 2), model.count);
    
    _ = counter.update(&model, .reset, &ctx.ctx);
    try testing.expectEqual(@as(i32, 0), model.count);
}

test "Counter view renders" {
    var ctx = try TestContext.init();
    defer ctx.deinit();
    
    const model = counter.init();
    const element = counter.view(&model, &ctx.ctx);
    
    try testing.expect(element == .element);
}
```

### Todo App Test

```zig
// tests/integration/todo_test.zig
const std = @import("std");
const testing = std.testing;
const todo = @import("../../examples/todo/src/main.zig");
const TestContext = @import("zui").TestContext;

test "Todo initialization" {
    var model = try todo.Model.init(testing.allocator);
    defer model.deinit();
    
    try testing.expectEqual(@as(usize, 0), model.todos.items.len);
    try testing.expectEqual(todo.Filter.all, model.filter);
}

test "Add todo" {
    var ctx = try TestContext.init();
    defer ctx.deinit();
    
    var model = try todo.Model.init(testing.allocator);
    defer model.deinit();
    
    model.input = "Test todo";
    _ = todo.update(&model, .add_todo, &ctx.ctx);
    
    try testing.expectEqual(@as(usize, 1), model.todos.items.len);
    try testing.expectEqualStrings("Test todo", model.todos.items[0].text);
    try testing.expectEqual(false, model.todos.items[0].completed);
}

test "Toggle todo" {
    var ctx = try TestContext.init();
    defer ctx.deinit();
    
    var model = try todo.Model.init(testing.allocator);
    defer model.deinit();
    
    try model.todos.append(.{
        .id = 1,
        .text = "Test",
        .completed = false,
    });
    
    _ = todo.update(&model, .{ .toggle_todo = 1 }, &ctx.ctx);
    
    try testing.expectEqual(true, model.todos.items[0].completed);
}

test "Delete todo" {
    var ctx = try TestContext.init();
    defer ctx.deinit();
    
    var model = try todo.Model.init(testing.allocator);
    defer model.deinit();
    
    try model.todos.append(.{ .id = 1, .text = "Test", .completed = false });
    try model.todos.append(.{ .id = 2, .text = "Test 2", .completed = false });
    
    _ = todo.update(&model, .{ .delete_todo = 1 }, &ctx.ctx);
    
    try testing.expectEqual(@as(usize, 1), model.todos.items.len);
    try testing.expectEqual(@as(u32, 2), model.todos.items[0].id);
}

test "Clear completed" {
    var ctx = try TestContext.init();
    defer ctx.deinit();
    
    var model = try todo.Model.init(testing.allocator);
    defer model.deinit();
    
    try model.todos.append(.{ .id = 1, .text = "Active", .completed = false });
    try model.todos.append(.{ .id = 2, .text = "Done", .completed = true });
    try model.todos.append(.{ .id = 3, .text = "Also done", .completed = true });
    
    _ = todo.update(&model, .clear_completed, &ctx.ctx);
    
    try testing.expectEqual(@as(usize, 1), model.todos.items.len);
    try testing.expectEqual(@as(u32, 1), model.todos.items[0].id);
}
```

---

## BENCHMARKS

### VDOM Benchmark

```zig
// bench/vdom_bench.zig
const std = @import("std");
const Element = @import("zui").vdom.Element;
const diff = @import("zui").vdom.diff;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const iterations = 10000;
    
    // Benchmark: Element creation
    {
        var timer = try std.time.Timer.start();
        var i: usize = 0;
        while (i < iterations) : (i += 1) {
            const element = Element(u32).element("div", null, .{}, &.{});
            _ = element;
        }
        const elapsed = timer.read();
        std.debug.print("Element creation: {d} ns/op\n", .{elapsed / iterations});
    }
    
    // Benchmark: Diff identical elements
    {
        const old = Element(u32).text("Hello");
        const new = Element(u32).text("Hello");
        
        var timer = try std.time.Timer.start();
        var i: usize = 0;
        while (i < iterations) : (i += 1) {
            const patches = try diff(u32, allocator, old, new, 0);
            allocator.free(patches);
        }
        const elapsed = timer.read();
        std.debug.print("Diff identical: {d} ns/op\n", .{elapsed / iterations});
    }
    
    // Benchmark: Diff with changes
    {
        const old = Element(u32).text("Hello");
        const new = Element(u32).text("World");
        
        var timer = try std.time.Timer.start();
        var i: usize = 0;
        while (i < iterations) : (i += 1) {
            const patches = try diff(u32, allocator, old, new, 0);
            allocator.free(patches);
        }
        const elapsed = timer.read();
        std.debug.print("Diff with changes: {d} ns/op\n", .{elapsed / iterations});
    }
}
```

### CSS Benchmark

```zig
// bench/css_bench.zig
const std = @import("std");
const StyleRule = @import("zui").css.StyleRule;
const CssBuilder = @import("zui").css.CssBuilder;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    
    const iterations = 10000;
    
    // Benchmark: Rule to CSS conversion
    {
        const rule = StyleRule{
            .display = .flex,
            .flex_direction = .column,
            .gap = .{ .rem = 1 },
            .padding = .{ .rem = 2 },
        };
        
        var timer = try std.time.Timer.start();
        var i: usize = 0;
        while (i < iterations) : (i += 1) {
            const css = try rule.toCss(allocator);
            allocator.free(css);
        }
        const elapsed = timer.read();
        std.debug.print("Rule to CSS: {d} ns/op\n", .{elapsed / iterations});
    }
    
    // Benchmark: Atomic class generation
    {
        var builder = CssBuilder.init(allocator);
        defer builder.deinit();
        
        const rule = StyleRule{
            .display = .flex,
            .gap = .{ .rem = 1 },
        };
        
        var timer = try std.time.Timer.start();
        var i: usize = 0;
        while (i < iterations) : (i += 1) {
            const classes = try builder.build(rule);
            _ = classes;
        }
        const elapsed = timer.read();
        std.debug.print("Atomic class gen: {d} ns/op\n", .{elapsed / iterations});
        
        const stats = builder.getStats();
        std.debug.print("Reuse factor: {d:.2}x\n", .{stats.avg_reuse});
    }
}
```

---

## CONTINUOUS TESTING

### Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "Running tests..."
zig build test

if [ $? -ne 0 ]; then
    echo "Tests failed. Commit aborted."
    exit 1
fi

echo "✓ All tests passed"
```

### CI Configuration

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Zig
      uses: goto-bus-stop/setup-zig@v2
      with:
        version: 0.15.x
    
    - name: Run unit tests
      run: zig build test
    
    - name: Run benchmarks
      run: zig build bench
    
    - name: Check coverage
      run: |
        zig build test -Dcoverage
        # Process coverage data
```

---

## COVERAGE TARGETS

| Module | Target | Current |
|--------|--------|---------|
| Core | 90% | TBD |
| VDOM | 95% | TBD |
| CSS | 90% | TBD |
| Router | 90% | TBD |
| Effects | 85% | TBD |
| Overall | 90% | TBD |

---

## CONCLUSION

The ZUI test suite ensures quality, prevents regressions, and validates performance. It includes unit tests, integration tests, and benchmarks.

**Links:**
- [← Previous: Examples](16-examples.md)
- [↑ Back to index](../README.md)
