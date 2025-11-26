# 24 - HEAD MANAGEMENT

**Document:** Dynamic Document Head Management
**Version:** 6.0.0
**Status:** Final
**Dependencies:** 03-virtual-dom.md, 18-fermyon-spin.md

---

## OVERVIEW

Head Management provides a declarative way to manage the document `<head>` section, including `<title>`, `<meta>`, `<link>`, and `<script>` tags. This is essential for:

- **SEO:** Dynamic meta descriptions, Open Graph tags, canonical URLs
- **Social Sharing:** Twitter cards, Facebook previews
- **Accessibility:** Document language, charset
- **Performance:** Preloading resources, prefetching
- **PWA:** Manifest, theme color, icons

---

## ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────┐
│                    HEAD MANAGEMENT SYSTEM                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Component Tree:              Document Head:                 │
│  ┌─────────────┐              ┌─────────────────────────┐   │
│  │    App      │              │ <head>                  │   │
│  │ ┌─────────┐ │              │   <title>Page</title>   │   │
│  │ │ Title() │─┼──────────────▶   <meta name="desc">   │   │
│  │ └─────────┘ │              │   <meta property="og">  │   │
│  │ ┌─────────┐ │              │   <link rel="canon">    │   │
│  │ │ Meta()  │─┼──────────────▶ </head>                 │   │
│  │ └─────────┘ │              └─────────────────────────┘   │
│  │ ┌─────────┐ │                                            │
│  │ │ Page    │ │   Nested components can also add tags:    │
│  │ │┌───────┐│ │   - Inner Title() overrides outer         │
│  │ ││Meta() ││─┼──▶ - Meta tags merge by key               │
│  │ │└───────┘│ │   - Links deduplicate by rel+href         │
│  │ └─────────┘ │                                            │
│  └─────────────┘                                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## HEAD ELEMENTS

### Title Component

```zig
// src/head/title.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Set the document title
pub fn Title(
    comptime Msg: type,
    ctx: *AppContext,
    title: []const u8,
) vdom.Element(Msg) {
    return .{
        .head_element = .{
            .tag = .title,
            .content = title,
        },
    };
}

/// Set title with template
pub fn TitleTemplate(
    comptime Msg: type,
    ctx: *AppContext,
    config: TitleConfig,
) vdom.Element(Msg) {
    const full_title = if (config.template) |template|
        std.fmt.allocPrint(
            ctx.frame_arena.allocator(),
            template,
            .{config.title},
        ) catch config.title
    else
        config.title;

    return .{
        .head_element = .{
            .tag = .title,
            .content = full_title,
        },
    };
}

pub const TitleConfig = struct {
    title: []const u8,
    /// Template with {s} placeholder, e.g., "{s} | My Site"
    template: ?[]const u8 = null,
};
```

### Meta Component

```zig
// src/head/meta.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Add a meta tag to document head
pub fn Meta(
    comptime Msg: type,
    ctx: *AppContext,
    config: MetaConfig,
) vdom.Element(Msg) {
    return .{
        .head_element = .{
            .tag = .meta,
            .attrs = buildMetaAttrs(ctx, config),
        },
    };
}

pub const MetaConfig = union(enum) {
    /// <meta name="..." content="...">
    name: struct {
        name: []const u8,
        content: []const u8,
    },

    /// <meta property="..." content="..."> (Open Graph)
    property: struct {
        property: []const u8,
        content: []const u8,
    },

    /// <meta http-equiv="..." content="...">
    http_equiv: struct {
        http_equiv: []const u8,
        content: []const u8,
    },

    /// <meta charset="...">
    charset: []const u8,
};

fn buildMetaAttrs(ctx: *AppContext, config: MetaConfig) []const Attr {
    const allocator = ctx.frame_arena.allocator();

    return switch (config) {
        .name => |n| allocator.dupe(Attr, &.{
            .{ .key = "name", .value = n.name },
            .{ .key = "content", .value = n.content },
        }) catch &.{},
        .property => |p| allocator.dupe(Attr, &.{
            .{ .key = "property", .value = p.property },
            .{ .key = "content", .value = p.content },
        }) catch &.{},
        .http_equiv => |h| allocator.dupe(Attr, &.{
            .{ .key = "http-equiv", .value = h.http_equiv },
            .{ .key = "content", .value = h.content },
        }) catch &.{},
        .charset => |c| allocator.dupe(Attr, &.{
            .{ .key = "charset", .value = c },
        }) catch &.{},
    };
}

/// Common meta tag helpers
pub const meta = struct {
    /// <meta name="description" content="...">
    pub fn description(
        comptime Msg: type,
        ctx: *AppContext,
        content: []const u8,
    ) vdom.Element(Msg) {
        return Meta(Msg, ctx, .{ .name = .{
            .name = "description",
            .content = content,
        } });
    }

    /// <meta name="keywords" content="...">
    pub fn keywords(
        comptime Msg: type,
        ctx: *AppContext,
        content: []const u8,
    ) vdom.Element(Msg) {
        return Meta(Msg, ctx, .{ .name = .{
            .name = "keywords",
            .content = content,
        } });
    }

    /// <meta name="author" content="...">
    pub fn author(
        comptime Msg: type,
        ctx: *AppContext,
        content: []const u8,
    ) vdom.Element(Msg) {
        return Meta(Msg, ctx, .{ .name = .{
            .name = "author",
            .content = content,
        } });
    }

    /// <meta name="viewport" content="...">
    pub fn viewport(
        comptime Msg: type,
        ctx: *AppContext,
        content: []const u8,
    ) vdom.Element(Msg) {
        return Meta(Msg, ctx, .{ .name = .{
            .name = "viewport",
            .content = content,
        } });
    }

    /// <meta name="robots" content="...">
    pub fn robots(
        comptime Msg: type,
        ctx: *AppContext,
        config: RobotsConfig,
    ) vdom.Element(Msg) {
        var parts: [6][]const u8 = undefined;
        var count: usize = 0;

        if (config.index) {
            parts[count] = "index";
            count += 1;
        } else if (config.noindex) {
            parts[count] = "noindex";
            count += 1;
        }

        if (config.follow) {
            parts[count] = "follow";
            count += 1;
        } else if (config.nofollow) {
            parts[count] = "nofollow";
            count += 1;
        }

        if (config.noarchive) {
            parts[count] = "noarchive";
            count += 1;
        }

        if (config.nosnippet) {
            parts[count] = "nosnippet";
            count += 1;
        }

        const content = std.mem.join(ctx.frame_arena.allocator(), ", ", parts[0..count]) catch "index, follow";

        return Meta(Msg, ctx, .{ .name = .{
            .name = "robots",
            .content = content,
        } });
    }

    /// <meta name="theme-color" content="...">
    pub fn themeColor(
        comptime Msg: type,
        ctx: *AppContext,
        color: []const u8,
    ) vdom.Element(Msg) {
        return Meta(Msg, ctx, .{ .name = .{
            .name = "theme-color",
            .content = color,
        } });
    }
};

pub const RobotsConfig = struct {
    index: bool = true,
    noindex: bool = false,
    follow: bool = true,
    nofollow: bool = false,
    noarchive: bool = false,
    nosnippet: bool = false,
};
```

### Open Graph Meta Tags

```zig
// src/head/og.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");
const Meta = @import("meta.zig").Meta;

/// Open Graph meta tags for social sharing
pub fn OpenGraph(
    comptime Msg: type,
    ctx: *AppContext,
    config: OpenGraphConfig,
) vdom.Element(Msg) {
    var children: [10]vdom.Element(Msg) = undefined;
    var count: usize = 0;

    // Required properties
    children[count] = Meta(Msg, ctx, .{ .property = .{ .property = "og:title", .content = config.title } });
    count += 1;

    children[count] = Meta(Msg, ctx, .{ .property = .{ .property = "og:type", .content = @tagName(config.og_type) } });
    count += 1;

    if (config.url) |url| {
        children[count] = Meta(Msg, ctx, .{ .property = .{ .property = "og:url", .content = url } });
        count += 1;
    }

    if (config.image) |image| {
        children[count] = Meta(Msg, ctx, .{ .property = .{ .property = "og:image", .content = image } });
        count += 1;
    }

    // Optional properties
    if (config.description) |desc| {
        children[count] = Meta(Msg, ctx, .{ .property = .{ .property = "og:description", .content = desc } });
        count += 1;
    }

    if (config.site_name) |name| {
        children[count] = Meta(Msg, ctx, .{ .property = .{ .property = "og:site_name", .content = name } });
        count += 1;
    }

    if (config.locale) |locale| {
        children[count] = Meta(Msg, ctx, .{ .property = .{ .property = "og:locale", .content = locale } });
        count += 1;
    }

    return .{
        .fragment = ctx.frame_arena.allocator().dupe(vdom.Element(Msg), children[0..count]) catch &.{},
    };
}

pub const OpenGraphConfig = struct {
    title: []const u8,
    og_type: OgType = .website,
    url: ?[]const u8 = null,
    image: ?[]const u8 = null,
    description: ?[]const u8 = null,
    site_name: ?[]const u8 = null,
    locale: ?[]const u8 = null,
};

pub const OgType = enum {
    website,
    article,
    book,
    profile,
    music_song,
    music_album,
    video_movie,
    video_episode,
};
```

### Twitter Card Meta Tags

```zig
// src/head/twitter.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");
const Meta = @import("meta.zig").Meta;

/// Twitter Card meta tags
pub fn TwitterCard(
    comptime Msg: type,
    ctx: *AppContext,
    config: TwitterCardConfig,
) vdom.Element(Msg) {
    var children: [8]vdom.Element(Msg) = undefined;
    var count: usize = 0;

    // Card type
    children[count] = Meta(Msg, ctx, .{ .name = .{
        .name = "twitter:card",
        .content = @tagName(config.card),
    } });
    count += 1;

    // Title
    children[count] = Meta(Msg, ctx, .{ .name = .{
        .name = "twitter:title",
        .content = config.title,
    } });
    count += 1;

    // Optional properties
    if (config.description) |desc| {
        children[count] = Meta(Msg, ctx, .{ .name = .{
            .name = "twitter:description",
            .content = desc,
        } });
        count += 1;
    }

    if (config.image) |image| {
        children[count] = Meta(Msg, ctx, .{ .name = .{
            .name = "twitter:image",
            .content = image,
        } });
        count += 1;
    }

    if (config.site) |site| {
        children[count] = Meta(Msg, ctx, .{ .name = .{
            .name = "twitter:site",
            .content = site,
        } });
        count += 1;
    }

    if (config.creator) |creator| {
        children[count] = Meta(Msg, ctx, .{ .name = .{
            .name = "twitter:creator",
            .content = creator,
        } });
        count += 1;
    }

    return .{
        .fragment = ctx.frame_arena.allocator().dupe(vdom.Element(Msg), children[0..count]) catch &.{},
    };
}

pub const TwitterCardConfig = struct {
    card: CardType = .summary,
    title: []const u8,
    description: ?[]const u8 = null,
    image: ?[]const u8 = null,
    site: ?[]const u8 = null, // @username
    creator: ?[]const u8 = null, // @username
};

pub const CardType = enum {
    summary,
    summary_large_image,
    app,
    player,
};
```

### Link Component

```zig
// src/head/link.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Add a link tag to document head
pub fn Link(
    comptime Msg: type,
    ctx: *AppContext,
    config: LinkConfig,
) vdom.Element(Msg) {
    return .{
        .head_element = .{
            .tag = .link,
            .attrs = buildLinkAttrs(ctx, config),
        },
    };
}

pub const LinkConfig = struct {
    rel: LinkRel,
    href: []const u8,
    type_attr: ?[]const u8 = null,
    sizes: ?[]const u8 = null,
    media: ?[]const u8 = null,
    crossorigin: ?CrossOrigin = null,
    hreflang: ?[]const u8 = null,
    as_attr: ?[]const u8 = null, // for preload
};

pub const LinkRel = enum {
    stylesheet,
    canonical,
    icon,
    apple_touch_icon,
    manifest,
    preload,
    prefetch,
    preconnect,
    dns_prefetch,
    alternate,
    author,
    license,
    prev,
    next,

    pub fn toString(self: LinkRel) []const u8 {
        return switch (self) {
            .apple_touch_icon => "apple-touch-icon",
            .dns_prefetch => "dns-prefetch",
            else => @tagName(self),
        };
    }
};

pub const CrossOrigin = enum {
    anonymous,
    use_credentials,
};

fn buildLinkAttrs(ctx: *AppContext, config: LinkConfig) []const Attr {
    const allocator = ctx.frame_arena.allocator();
    var attrs = std.ArrayList(Attr).init(allocator);

    attrs.append(.{ .key = "rel", .value = config.rel.toString() }) catch {};
    attrs.append(.{ .key = "href", .value = config.href }) catch {};

    if (config.type_attr) |t| {
        attrs.append(.{ .key = "type", .value = t }) catch {};
    }
    if (config.sizes) |s| {
        attrs.append(.{ .key = "sizes", .value = s }) catch {};
    }
    if (config.media) |m| {
        attrs.append(.{ .key = "media", .value = m }) catch {};
    }
    if (config.crossorigin) |c| {
        attrs.append(.{ .key = "crossorigin", .value = @tagName(c) }) catch {};
    }
    if (config.hreflang) |h| {
        attrs.append(.{ .key = "hreflang", .value = h }) catch {};
    }
    if (config.as_attr) |a| {
        attrs.append(.{ .key = "as", .value = a }) catch {};
    }

    return attrs.toOwnedSlice() catch &.{};
}

/// Common link helpers
pub const link = struct {
    /// <link rel="canonical" href="...">
    pub fn canonical(
        comptime Msg: type,
        ctx: *AppContext,
        href: []const u8,
    ) vdom.Element(Msg) {
        return Link(Msg, ctx, .{ .rel = .canonical, .href = href });
    }

    /// <link rel="icon" href="..." type="...">
    pub fn favicon(
        comptime Msg: type,
        ctx: *AppContext,
        href: []const u8,
        icon_type: ?[]const u8,
    ) vdom.Element(Msg) {
        return Link(Msg, ctx, .{
            .rel = .icon,
            .href = href,
            .type_attr = icon_type,
        });
    }

    /// <link rel="apple-touch-icon" sizes="..." href="...">
    pub fn appleTouchIcon(
        comptime Msg: type,
        ctx: *AppContext,
        href: []const u8,
        sizes: ?[]const u8,
    ) vdom.Element(Msg) {
        return Link(Msg, ctx, .{
            .rel = .apple_touch_icon,
            .href = href,
            .sizes = sizes,
        });
    }

    /// <link rel="manifest" href="...">
    pub fn manifest(
        comptime Msg: type,
        ctx: *AppContext,
        href: []const u8,
    ) vdom.Element(Msg) {
        return Link(Msg, ctx, .{ .rel = .manifest, .href = href });
    }

    /// <link rel="preload" href="..." as="...">
    pub fn preload(
        comptime Msg: type,
        ctx: *AppContext,
        href: []const u8,
        as_type: []const u8,
    ) vdom.Element(Msg) {
        return Link(Msg, ctx, .{
            .rel = .preload,
            .href = href,
            .as_attr = as_type,
        });
    }

    /// <link rel="preconnect" href="...">
    pub fn preconnect(
        comptime Msg: type,
        ctx: *AppContext,
        href: []const u8,
    ) vdom.Element(Msg) {
        return Link(Msg, ctx, .{ .rel = .preconnect, .href = href });
    }

    /// <link rel="alternate" hreflang="..." href="...">
    pub fn alternate(
        comptime Msg: type,
        ctx: *AppContext,
        href: []const u8,
        hreflang: []const u8,
    ) vdom.Element(Msg) {
        return Link(Msg, ctx, .{
            .rel = .alternate,
            .href = href,
            .hreflang = hreflang,
        });
    }

    /// <link rel="stylesheet" href="...">
    pub fn stylesheet(
        comptime Msg: type,
        ctx: *AppContext,
        href: []const u8,
    ) vdom.Element(Msg) {
        return Link(Msg, ctx, .{ .rel = .stylesheet, .href = href });
    }
};
```

### Script Component

```zig
// src/head/script.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Add a script tag to document head
pub fn Script(
    comptime Msg: type,
    ctx: *AppContext,
    config: ScriptConfig,
) vdom.Element(Msg) {
    return .{
        .head_element = .{
            .tag = .script,
            .attrs = buildScriptAttrs(ctx, config),
            .content = config.inline_content,
        },
    };
}

pub const ScriptConfig = struct {
    src: ?[]const u8 = null,
    inline_content: ?[]const u8 = null,
    script_type: ?[]const u8 = null, // "module", "application/json", etc.
    async_attr: bool = false,
    defer_attr: bool = false,
    crossorigin: ?CrossOrigin = null,
    integrity: ?[]const u8 = null,
    nomodule: bool = false,
    id: ?[]const u8 = null,
};

fn buildScriptAttrs(ctx: *AppContext, config: ScriptConfig) []const Attr {
    const allocator = ctx.frame_arena.allocator();
    var attrs = std.ArrayList(Attr).init(allocator);

    if (config.src) |s| {
        attrs.append(.{ .key = "src", .value = s }) catch {};
    }
    if (config.script_type) |t| {
        attrs.append(.{ .key = "type", .value = t }) catch {};
    }
    if (config.async_attr) {
        attrs.append(.{ .key = "async", .value = "" }) catch {};
    }
    if (config.defer_attr) {
        attrs.append(.{ .key = "defer", .value = "" }) catch {};
    }
    if (config.crossorigin) |c| {
        attrs.append(.{ .key = "crossorigin", .value = @tagName(c) }) catch {};
    }
    if (config.integrity) |i| {
        attrs.append(.{ .key = "integrity", .value = i }) catch {};
    }
    if (config.nomodule) {
        attrs.append(.{ .key = "nomodule", .value = "" }) catch {};
    }
    if (config.id) |id| {
        attrs.append(.{ .key = "id", .value = id }) catch {};
    }

    return attrs.toOwnedSlice() catch &.{};
}

/// JSON-LD structured data
pub fn JsonLd(
    comptime Msg: type,
    ctx: *AppContext,
    data: anytype,
) vdom.Element(Msg) {
    const json = std.json.stringifyAlloc(
        ctx.frame_arena.allocator(),
        data,
        .{},
    ) catch return .none;

    return Script(Msg, ctx, .{
        .script_type = "application/ld+json",
        .inline_content = json,
    });
}
```

---

## VDOM EXTENSION

### Head Element Type

```zig
// Addition to src/vdom/element.zig

pub fn Element(comptime Msg: type) type {
    return union(enum) {
        // ... existing variants ...

        /// Element that renders into document <head>
        head_element: HeadElement,

        /// Fragment (multiple elements without wrapper)
        fragment: []Element(Msg),
    };
}

pub const HeadElement = struct {
    tag: HeadTag,
    attrs: []const Attr = &.{},
    content: ?[]const u8 = null,
};

pub const HeadTag = enum {
    title,
    meta,
    link,
    script,
    style,
    base,
};

pub const Attr = struct {
    key: []const u8,
    value: []const u8,
};
```

---

## HEAD COLLECTOR

### Runtime Collection

```zig
// src/head/collector.zig
const std = @import("std");
const vdom = @import("../vdom/element.zig");

/// Collects head elements from the VDOM tree
pub const HeadCollector = struct {
    title: ?[]const u8 = null,
    metas: std.ArrayList(HeadElement),
    links: std.ArrayList(HeadElement),
    scripts: std.ArrayList(HeadElement),
    styles: std.ArrayList(HeadElement),

    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) HeadCollector {
        return .{
            .metas = std.ArrayList(HeadElement).init(allocator),
            .links = std.ArrayList(HeadElement).init(allocator),
            .scripts = std.ArrayList(HeadElement).init(allocator),
            .styles = std.ArrayList(HeadElement).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *HeadCollector) void {
        self.metas.deinit();
        self.links.deinit();
        self.scripts.deinit();
        self.styles.deinit();
    }

    /// Collect head elements from VDOM tree
    pub fn collect(self: *HeadCollector, comptime Msg: type, element: vdom.Element(Msg)) void {
        switch (element) {
            .head_element => |head| {
                switch (head.tag) {
                    .title => self.title = head.content,
                    .meta => self.addMeta(head),
                    .link => self.addLink(head),
                    .script => self.scripts.append(head) catch {},
                    .style => self.styles.append(head) catch {},
                    .base => {}, // Handle base tag
                }
            },
            .element => |el| {
                for (el.children) |child| {
                    self.collect(Msg, child);
                }
            },
            .fragment => |children| {
                for (children) |child| {
                    self.collect(Msg, child);
                }
            },
            .portal => |p| {
                for (p.children) |child| {
                    self.collect(Msg, child);
                }
            },
            else => {},
        }
    }

    /// Add meta, deduplicating by name/property
    fn addMeta(self: *HeadCollector, meta: HeadElement) void {
        const key = getMetaKey(meta.attrs);

        // Check for existing meta with same key
        for (self.metas.items, 0..) |existing, i| {
            if (std.mem.eql(u8, getMetaKey(existing.attrs), key)) {
                // Replace existing (later declarations win)
                self.metas.items[i] = meta;
                return;
            }
        }

        self.metas.append(meta) catch {};
    }

    /// Add link, deduplicating by rel+href
    fn addLink(self: *HeadCollector, link_elem: HeadElement) void {
        const rel = getAttr(link_elem.attrs, "rel") orelse "";
        const href = getAttr(link_elem.attrs, "href") orelse "";

        // Check for existing link with same rel+href
        for (self.links.items) |existing| {
            const existing_rel = getAttr(existing.attrs, "rel") orelse "";
            const existing_href = getAttr(existing.attrs, "href") orelse "";

            if (std.mem.eql(u8, rel, existing_rel) and
                std.mem.eql(u8, href, existing_href))
            {
                // Already exists, skip
                return;
            }
        }

        self.links.append(link_elem) catch {};
    }

    fn getMetaKey(attrs: []const Attr) []const u8 {
        return getAttr(attrs, "name") orelse
            getAttr(attrs, "property") orelse
            getAttr(attrs, "http-equiv") orelse
            getAttr(attrs, "charset") orelse "";
    }

    fn getAttr(attrs: []const Attr, key: []const u8) ?[]const u8 {
        for (attrs) |attr| {
            if (std.mem.eql(u8, attr.key, key)) {
                return attr.value;
            }
        }
        return null;
    }
};
```

---

## SSR INTEGRATION

### Server-Side Head Rendering

```zig
// src/ssr/head_render.zig
const std = @import("std");
const HeadCollector = @import("../head/collector.zig").HeadCollector;

/// Render collected head elements to HTML string
pub fn renderHead(collector: *const HeadCollector, writer: anytype) !void {
    // Title
    if (collector.title) |title| {
        try writer.print("<title>{s}</title>\n", .{escapeHtml(title)});
    }

    // Meta tags
    for (collector.metas.items) |meta| {
        try writer.writeAll("<meta");
        for (meta.attrs) |attr| {
            try writer.print(" {s}=\"{s}\"", .{ attr.key, escapeAttr(attr.value) });
        }
        try writer.writeAll(">\n");
    }

    // Link tags
    for (collector.links.items) |link_elem| {
        try writer.writeAll("<link");
        for (link_elem.attrs) |attr| {
            try writer.print(" {s}=\"{s}\"", .{ attr.key, escapeAttr(attr.value) });
        }
        try writer.writeAll(">\n");
    }

    // Scripts
    for (collector.scripts.items) |script| {
        try writer.writeAll("<script");
        for (script.attrs) |attr| {
            if (attr.value.len == 0) {
                try writer.print(" {s}", .{attr.key});
            } else {
                try writer.print(" {s}=\"{s}\"", .{ attr.key, escapeAttr(attr.value) });
            }
        }
        try writer.writeAll(">");
        if (script.content) |content| {
            try writer.writeAll(content);
        }
        try writer.writeAll("</script>\n");
    }

    // Styles
    for (collector.styles.items) |style| {
        try writer.writeAll("<style");
        for (style.attrs) |attr| {
            try writer.print(" {s}=\"{s}\"", .{ attr.key, escapeAttr(attr.value) });
        }
        try writer.writeAll(">");
        if (style.content) |content| {
            try writer.writeAll(content);
        }
        try writer.writeAll("</style>\n");
    }
}
```

---

## JS RUNTIME INTEGRATION

```javascript
// www/zui.js - Head management

const ZuiHead = {
    // Track current head elements for diffing
    currentTitle: null,
    currentMetas: new Map(), // key -> element
    currentLinks: new Map(), // rel+href -> element

    // Update document title
    setTitle(title) {
        if (this.currentTitle !== title) {
            document.title = title;
            this.currentTitle = title;
        }
    },

    // Update or add meta tag
    setMeta(key, attrs) {
        const existing = this.currentMetas.get(key);

        if (existing) {
            // Update existing
            for (const [name, value] of Object.entries(attrs)) {
                existing.setAttribute(name, value);
            }
        } else {
            // Create new
            const meta = document.createElement('meta');
            for (const [name, value] of Object.entries(attrs)) {
                meta.setAttribute(name, value);
            }
            document.head.appendChild(meta);
            this.currentMetas.set(key, meta);
        }
    },

    // Remove meta tag
    removeMeta(key) {
        const existing = this.currentMetas.get(key);
        if (existing) {
            existing.remove();
            this.currentMetas.delete(key);
        }
    },

    // Update or add link tag
    setLink(rel, href, attrs) {
        const key = `${rel}:${href}`;
        const existing = this.currentLinks.get(key);

        if (existing) {
            // Update existing
            for (const [name, value] of Object.entries(attrs)) {
                existing.setAttribute(name, value);
            }
        } else {
            // Create new
            const link = document.createElement('link');
            link.rel = rel;
            link.href = href;
            for (const [name, value] of Object.entries(attrs)) {
                link.setAttribute(name, value);
            }
            document.head.appendChild(link);
            this.currentLinks.set(key, link);
        }
    },

    // Remove link tag
    removeLink(rel, href) {
        const key = `${rel}:${href}`;
        const existing = this.currentLinks.get(key);
        if (existing) {
            existing.remove();
            this.currentLinks.delete(key);
        }
    },

    // Apply head updates from WASM
    applyUpdates(updates) {
        if (updates.title !== undefined) {
            this.setTitle(updates.title);
        }

        if (updates.metas) {
            for (const [key, attrs] of Object.entries(updates.metas)) {
                if (attrs === null) {
                    this.removeMeta(key);
                } else {
                    this.setMeta(key, attrs);
                }
            }
        }

        if (updates.links) {
            for (const [key, attrs] of Object.entries(updates.links)) {
                if (attrs === null) {
                    const [rel, href] = key.split(':');
                    this.removeLink(rel, href);
                } else {
                    const [rel, href] = key.split(':');
                    this.setLink(rel, href, attrs);
                }
            }
        }
    },
};
```

---

## USAGE EXAMPLES

### Basic Page with SEO

```zig
const zui = @import("zui");
const h = zui.html;
const head = zui.head;

pub fn BlogPost(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    const post = model.post;

    return h.fragment(&.{
        // Head elements
        head.Title(Msg, ctx, post.title),
        head.meta.description(Msg, ctx, post.excerpt),
        head.link.canonical(Msg, ctx, post.url),

        // Open Graph
        head.OpenGraph(Msg, ctx, .{
            .title = post.title,
            .og_type = .article,
            .description = post.excerpt,
            .image = post.cover_image,
            .url = post.url,
        }),

        // Twitter Card
        head.TwitterCard(Msg, ctx, .{
            .card = .summary_large_image,
            .title = post.title,
            .description = post.excerpt,
            .image = post.cover_image,
        }),

        // JSON-LD structured data
        head.JsonLd(Msg, ctx, .{
            .@"@context" = "https://schema.org",
            .@"@type" = "BlogPosting",
            .headline = post.title,
            .description = post.excerpt,
            .image = post.cover_image,
            .datePublished = post.published_at,
            .author = .{
                .@"@type" = "Person",
                .name = post.author.name,
            },
        }),

        // Body content
        h.article(ctx, .{ .class = "blog-post" }, &.{
            h.h1(ctx, .{}, &.{ h.text(ctx, post.title) }),
            h.div(ctx, .{
                .class = "content",
            }, &.{
                h.rawHtml(ctx, post.content),
            }),
        }),
    });
}
```

### Multi-language Site

```zig
pub fn Page(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return h.fragment(&.{
        head.Title(Msg, ctx, model.i18n.get("page.title")),
        head.meta.description(Msg, ctx, model.i18n.get("page.description")),

        // Alternate language versions
        head.link.alternate(Msg, ctx, "/en" ++ model.path, "en"),
        head.link.alternate(Msg, ctx, "/es" ++ model.path, "es"),
        head.link.alternate(Msg, ctx, "/fr" ++ model.path, "fr"),

        // x-default for language selector
        head.Link(Msg, ctx, .{
            .rel = .alternate,
            .href = model.path,
            .hreflang = "x-default",
        }),

        // ... page content
    });
}
```

### Performance Optimizations

```zig
pub fn AppShell(ctx: *zui.AppContext, model: *const Model) zui.Element(Msg) {
    return h.fragment(&.{
        // Preconnect to API
        head.link.preconnect(Msg, ctx, "https://api.example.com"),
        head.link.preconnect(Msg, ctx, "https://cdn.example.com"),

        // DNS prefetch for third-party resources
        head.Link(Msg, ctx, .{
            .rel = .dns_prefetch,
            .href = "https://fonts.googleapis.com",
        }),

        // Preload critical resources
        head.link.preload(Msg, ctx, "/fonts/main.woff2", "font"),
        head.link.preload(Msg, ctx, "/images/hero.webp", "image"),

        // ... page content
    });
}
```

---

## CONCLUSION

Head Management provides a powerful, declarative API for controlling the document `<head>`. The system supports both client-side SPA updates and server-side rendering for SSR/SSG modes, ensuring proper SEO and social sharing across all rendering strategies.

**Links:**
- [← Previous: Portals](23-portals.md)
- [→ Next: Transitions](25-transitions.md)
- [↑ Back to index](../README.md)
