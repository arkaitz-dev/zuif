# 27. Spin SDK for Zig

## Overview

This specification defines a native Zig SDK for Fermyon Spin, providing idiomatic Zig bindings for Spin's WebAssembly component capabilities. The SDK enables zuif applications to interact with Spin's host features using Zig's type safety, error handling, and memory management patterns.

## Motivation

Currently, there is no official Fermyon SDK for Zig. The existing options are:
- **wit-bindgen**: Generates C code, requiring `@cImport` workarounds
- **zig-spin**: Third-party, work-in-progress, incomplete coverage

A native Zig SDK provides:
- Idiomatic Zig APIs with proper error unions and allocator support
- Compile-time type safety
- Zero-cost abstractions over WASM host calls
- Seamless integration with zuif's architecture
- Clear memory ownership semantics

## Modular Architecture

The SDK follows a phased, modular approach:

```
spin-sdk/
├── core.zig           # Core types and utilities
├── http/
│   ├── inbound.zig    # HTTP trigger handling
│   └── outbound.zig   # Outgoing HTTP requests
├── kv.zig             # Key-Value Store
├── config.zig         # Configuration and Variables
├── sqlite.zig         # SQLite database (Phase 2)
└── future/            # Phase 3 modules
    ├── redis.zig
    ├── postgres.zig
    ├── mysql.zig
    └── llm.zig
```

### Implementation Phases

| Phase | Modules | Priority | Complexity |
|-------|---------|----------|------------|
| 1 (MVP) | HTTP Inbound/Outbound, KV, Config | Critical | ~1000 lines |
| 2 | SQLite | High | ~400 lines |
| 3 | Redis, PostgreSQL, MySQL, LLM | Future | ~1500 lines |

---

## Phase 1: MVP

### 27.1 Core Types

```zig
// src/spin/core.zig

const std = @import("std");

/// Spin SDK error types
pub const SpinError = error{
    /// Store not found or not configured
    StoreNotFound,
    /// Key not found in store
    KeyNotFound,
    /// Invalid key format
    InvalidKey,
    /// Value too large
    ValueTooLarge,
    /// Store is read-only
    ReadOnly,
    /// Configuration key not found
    ConfigNotFound,
    /// Invalid configuration value
    InvalidConfig,
    /// HTTP request failed
    HttpRequestFailed,
    /// Invalid URL
    InvalidUrl,
    /// Connection timeout
    Timeout,
    /// Host function not available
    HostNotAvailable,
    /// Memory allocation failed
    OutOfMemory,
    /// Invalid UTF-8 encoding
    InvalidUtf8,
    /// Database error
    DatabaseError,
    /// Query execution failed
    QueryFailed,
    /// Permission denied
    PermissionDenied,
    /// Resource exhausted
    ResourceExhausted,
    /// Internal error
    InternalError,
};

/// Result type for SDK operations
pub fn Result(comptime T: type) type {
    return union(enum) {
        ok: T,
        err: SpinError,

        pub fn unwrap(self: @This()) SpinError!T {
            return switch (self) {
                .ok => |v| v,
                .err => |e| e,
            };
        }

        pub fn unwrapOr(self: @This(), default: T) T {
            return switch (self) {
                .ok => |v| v,
                .err => default,
            };
        }
    };
}

/// Byte slice with ownership tracking
pub const OwnedBytes = struct {
    data: []u8,
    allocator: std.mem.Allocator,

    pub fn deinit(self: *OwnedBytes) void {
        self.allocator.free(self.data);
    }

    pub fn toSlice(self: OwnedBytes) []const u8 {
        return self.data;
    }
};

/// String with ownership tracking
pub const OwnedString = struct {
    data: []u8,
    allocator: std.mem.Allocator,

    pub fn deinit(self: *OwnedString) void {
        self.allocator.free(self.data);
    }

    pub fn toSlice(self: OwnedString) []const u8 {
        return self.data;
    }
};
```

### 27.2 HTTP Inbound (Triggers)

```zig
// src/spin/http/inbound.zig

const std = @import("std");
const core = @import("../core.zig");

/// HTTP method
pub const Method = enum {
    GET,
    POST,
    PUT,
    DELETE,
    PATCH,
    HEAD,
    OPTIONS,
    TRACE,
    CONNECT,

    pub fn fromString(s: []const u8) ?Method {
        const map = std.ComptimeStringMap(Method, .{
            .{ "GET", .GET },
            .{ "POST", .POST },
            .{ "PUT", .PUT },
            .{ "DELETE", .DELETE },
            .{ "PATCH", .PATCH },
            .{ "HEAD", .HEAD },
            .{ "OPTIONS", .OPTIONS },
            .{ "TRACE", .TRACE },
            .{ "CONNECT", .CONNECT },
        });
        return map.get(s);
    }

    pub fn toString(self: Method) []const u8 {
        return @tagName(self);
    }
};

/// HTTP status codes
pub const StatusCode = enum(u16) {
    // 2xx Success
    ok = 200,
    created = 201,
    accepted = 202,
    no_content = 204,

    // 3xx Redirection
    moved_permanently = 301,
    found = 302,
    see_other = 303,
    not_modified = 304,
    temporary_redirect = 307,
    permanent_redirect = 308,

    // 4xx Client Errors
    bad_request = 400,
    unauthorized = 401,
    forbidden = 403,
    not_found = 404,
    method_not_allowed = 405,
    conflict = 409,
    gone = 410,
    unprocessable_entity = 422,
    too_many_requests = 429,

    // 5xx Server Errors
    internal_server_error = 500,
    not_implemented = 501,
    bad_gateway = 502,
    service_unavailable = 503,
    gateway_timeout = 504,
    _,

    pub fn phrase(self: StatusCode) []const u8 {
        return switch (self) {
            .ok => "OK",
            .created => "Created",
            .no_content => "No Content",
            .bad_request => "Bad Request",
            .unauthorized => "Unauthorized",
            .forbidden => "Forbidden",
            .not_found => "Not Found",
            .internal_server_error => "Internal Server Error",
            else => "Unknown",
        };
    }
};

/// HTTP headers collection
pub const Headers = struct {
    entries: std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) Headers {
        return .{
            .entries = std.StringHashMap([]const u8).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Headers) void {
        self.entries.deinit();
    }

    pub fn get(self: *const Headers, name: []const u8) ?[]const u8 {
        // Case-insensitive lookup
        var lower_buf: [256]u8 = undefined;
        const lower = std.ascii.lowerString(&lower_buf, name);
        return self.entries.get(lower);
    }

    pub fn set(self: *Headers, name: []const u8, value: []const u8) !void {
        var lower_buf: [256]u8 = undefined;
        const lower = std.ascii.lowerString(&lower_buf, name);
        const key = try self.allocator.dupe(u8, lower);
        const val = try self.allocator.dupe(u8, value);
        try self.entries.put(key, val);
    }

    pub fn iterator(self: *const Headers) std.StringHashMap([]const u8).Iterator {
        return self.entries.iterator();
    }
};

/// Incoming HTTP request
pub const Request = struct {
    method: Method,
    uri: []const u8,
    path: []const u8,
    query: ?[]const u8,
    headers: Headers,
    body: ?[]const u8,
    allocator: std.mem.Allocator,

    /// Parse query parameters
    pub fn queryParams(self: *const Request) !QueryParams {
        return QueryParams.parse(self.allocator, self.query orelse "");
    }

    /// Get a specific header value
    pub fn header(self: *const Request, name: []const u8) ?[]const u8 {
        return self.headers.get(name);
    }

    /// Get content type
    pub fn contentType(self: *const Request) ?[]const u8 {
        return self.header("content-type");
    }

    /// Check if request accepts JSON
    pub fn acceptsJson(self: *const Request) bool {
        const accept = self.header("accept") orelse return false;
        return std.mem.indexOf(u8, accept, "application/json") != null;
    }

    /// Parse body as JSON
    pub fn json(self: *const Request, comptime T: type) !T {
        const body = self.body orelse return error.NoBody;
        return std.json.parseFromSlice(T, self.allocator, body, .{});
    }
};

/// Query parameters
pub const QueryParams = struct {
    params: std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,

    pub fn parse(allocator: std.mem.Allocator, query: []const u8) !QueryParams {
        var params = std.StringHashMap([]const u8).init(allocator);

        var iter = std.mem.splitScalar(u8, query, '&');
        while (iter.next()) |pair| {
            if (pair.len == 0) continue;

            if (std.mem.indexOf(u8, pair, "=")) |eq_pos| {
                const key = pair[0..eq_pos];
                const value = pair[eq_pos + 1 ..];
                try params.put(key, value);
            } else {
                try params.put(pair, "");
            }
        }

        return .{ .params = params, .allocator = allocator };
    }

    pub fn get(self: *const QueryParams, key: []const u8) ?[]const u8 {
        return self.params.get(key);
    }

    pub fn deinit(self: *QueryParams) void {
        self.params.deinit();
    }
};

/// HTTP response builder
pub const Response = struct {
    status: StatusCode,
    headers: Headers,
    body: ?[]const u8,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) Response {
        return .{
            .status = .ok,
            .headers = Headers.init(allocator),
            .body = null,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Response) void {
        self.headers.deinit();
    }

    /// Set status code
    pub fn setStatus(self: *Response, status: StatusCode) *Response {
        self.status = status;
        return self;
    }

    /// Set a header
    pub fn setHeader(self: *Response, name: []const u8, value: []const u8) !*Response {
        try self.headers.set(name, value);
        return self;
    }

    /// Set content type
    pub fn setContentType(self: *Response, content_type: []const u8) !*Response {
        return self.setHeader("content-type", content_type);
    }

    /// Set body
    pub fn setBody(self: *Response, body: []const u8) *Response {
        self.body = body;
        return self;
    }

    /// Set JSON body
    pub fn json(self: *Response, value: anytype) !*Response {
        const body = try std.json.stringifyAlloc(self.allocator, value, .{});
        _ = try self.setContentType("application/json");
        return self.setBody(body);
    }

    /// Set HTML body
    pub fn html(self: *Response, content: []const u8) !*Response {
        _ = try self.setContentType("text/html; charset=utf-8");
        return self.setBody(content);
    }

    /// Set text body
    pub fn text(self: *Response, content: []const u8) !*Response {
        _ = try self.setContentType("text/plain; charset=utf-8");
        return self.setBody(content);
    }

    /// Create redirect response
    pub fn redirect(self: *Response, location: []const u8, permanent: bool) !*Response {
        self.status = if (permanent) .permanent_redirect else .temporary_redirect;
        _ = try self.setHeader("location", location);
        return self;
    }
};

/// Convenience functions for common responses
pub const responses = struct {
    pub fn ok(allocator: std.mem.Allocator) Response {
        return Response.init(allocator);
    }

    pub fn notFound(allocator: std.mem.Allocator) !Response {
        var resp = Response.init(allocator);
        _ = resp.setStatus(.not_found);
        _ = try resp.text("Not Found");
        return resp;
    }

    pub fn badRequest(allocator: std.mem.Allocator, message: []const u8) !Response {
        var resp = Response.init(allocator);
        _ = resp.setStatus(.bad_request);
        _ = try resp.text(message);
        return resp;
    }

    pub fn internalError(allocator: std.mem.Allocator) !Response {
        var resp = Response.init(allocator);
        _ = resp.setStatus(.internal_server_error);
        _ = try resp.text("Internal Server Error");
        return resp;
    }

    pub fn jsonResponse(allocator: std.mem.Allocator, value: anytype) !Response {
        var resp = Response.init(allocator);
        _ = try resp.json(value);
        return resp;
    }
};

/// HTTP trigger handler type
pub const Handler = *const fn (*Request, std.mem.Allocator) anyerror!Response;
```

### 27.3 HTTP Outbound

```zig
// src/spin/http/outbound.zig

const std = @import("std");
const core = @import("../core.zig");
const inbound = @import("inbound.zig");

/// Outbound HTTP request builder
pub const OutboundRequest = struct {
    method: inbound.Method,
    url: []const u8,
    headers: inbound.Headers,
    body: ?[]const u8,
    timeout_ms: ?u32,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator, url: []const u8) OutboundRequest {
        return .{
            .method = .GET,
            .url = url,
            .headers = inbound.Headers.init(allocator),
            .body = null,
            .timeout_ms = null,
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *OutboundRequest) void {
        self.headers.deinit();
    }

    pub fn setMethod(self: *OutboundRequest, method: inbound.Method) *OutboundRequest {
        self.method = method;
        return self;
    }

    pub fn get(self: *OutboundRequest) *OutboundRequest {
        return self.setMethod(.GET);
    }

    pub fn post(self: *OutboundRequest) *OutboundRequest {
        return self.setMethod(.POST);
    }

    pub fn put(self: *OutboundRequest) *OutboundRequest {
        return self.setMethod(.PUT);
    }

    pub fn delete(self: *OutboundRequest) *OutboundRequest {
        return self.setMethod(.DELETE);
    }

    pub fn setHeader(self: *OutboundRequest, name: []const u8, value: []const u8) !*OutboundRequest {
        try self.headers.set(name, value);
        return self;
    }

    pub fn setBody(self: *OutboundRequest, body: []const u8) *OutboundRequest {
        self.body = body;
        return self;
    }

    pub fn json(self: *OutboundRequest, value: anytype) !*OutboundRequest {
        const body = try std.json.stringifyAlloc(self.allocator, value, .{});
        self.body = body;
        _ = try self.setHeader("content-type", "application/json");
        return self;
    }

    pub fn setTimeout(self: *OutboundRequest, ms: u32) *OutboundRequest {
        self.timeout_ms = ms;
        return self;
    }

    /// Send the request and get response
    pub fn send(self: *OutboundRequest) !OutboundResponse {
        return http_send(self);
    }
};

/// Outbound HTTP response
pub const OutboundResponse = struct {
    status: inbound.StatusCode,
    headers: inbound.Headers,
    body: core.OwnedBytes,
    allocator: std.mem.Allocator,

    pub fn deinit(self: *OutboundResponse) void {
        self.headers.deinit();
        self.body.deinit();
    }

    /// Check if response is successful (2xx)
    pub fn isOk(self: *const OutboundResponse) bool {
        const code = @intFromEnum(self.status);
        return code >= 200 and code < 300;
    }

    /// Get body as string
    pub fn text(self: *const OutboundResponse) []const u8 {
        return self.body.toSlice();
    }

    /// Parse body as JSON
    pub fn json(self: *const OutboundResponse, comptime T: type) !T {
        return std.json.parseFromSlice(T, self.allocator, self.body.data, .{});
    }
};

/// HTTP client for making multiple requests
pub const Client = struct {
    allocator: std.mem.Allocator,
    default_headers: inbound.Headers,
    base_url: ?[]const u8,
    default_timeout_ms: ?u32,

    pub fn init(allocator: std.mem.Allocator) Client {
        return .{
            .allocator = allocator,
            .default_headers = inbound.Headers.init(allocator),
            .base_url = null,
            .default_timeout_ms = null,
        };
    }

    pub fn deinit(self: *Client) void {
        self.default_headers.deinit();
    }

    pub fn setBaseUrl(self: *Client, url: []const u8) *Client {
        self.base_url = url;
        return self;
    }

    pub fn setDefaultHeader(self: *Client, name: []const u8, value: []const u8) !*Client {
        try self.default_headers.set(name, value);
        return self;
    }

    pub fn setDefaultTimeout(self: *Client, ms: u32) *Client {
        self.default_timeout_ms = ms;
        return self;
    }

    /// Create a new request
    pub fn request(self: *Client, path: []const u8) !OutboundRequest {
        const url = if (self.base_url) |base|
            try std.fmt.allocPrint(self.allocator, "{s}{s}", .{ base, path })
        else
            path;

        var req = OutboundRequest.init(self.allocator, url);

        // Copy default headers
        var iter = self.default_headers.iterator();
        while (iter.next()) |entry| {
            _ = try req.setHeader(entry.key_ptr.*, entry.value_ptr.*);
        }

        if (self.default_timeout_ms) |timeout| {
            _ = req.setTimeout(timeout);
        }

        return req;
    }

    /// Convenience method for GET request
    pub fn get(self: *Client, path: []const u8) !OutboundResponse {
        var req = try self.request(path);
        defer req.deinit();
        return req.send();
    }

    /// Convenience method for POST request with JSON body
    pub fn postJson(self: *Client, path: []const u8, body: anytype) !OutboundResponse {
        var req = try self.request(path);
        defer req.deinit();
        _ = req.post();
        _ = try req.json(body);
        return req.send();
    }
};

// Internal: actual HTTP send implementation (bridges to WASM host)
fn http_send(req: *OutboundRequest) !OutboundResponse {
    // This will be implemented via WASM host calls
    // The actual implementation will use wit-bindgen generated C code
    // or direct WASM imports
    _ = req;
    @panic("http_send not implemented - requires WASM host bindings");
}
```

### 27.4 Key-Value Store

```zig
// src/spin/kv.zig

const std = @import("std");
const core = @import("core.zig");

/// Key-Value store handle
pub const Store = struct {
    name: []const u8,
    handle: u32, // Internal handle from host
    allocator: std.mem.Allocator,

    /// Open a key-value store by name
    pub fn open(allocator: std.mem.Allocator, name: []const u8) !Store {
        const handle = try kv_open(name);
        return .{
            .name = name,
            .handle = handle,
            .allocator = allocator,
        };
    }

    /// Open the default store
    pub fn openDefault(allocator: std.mem.Allocator) !Store {
        return open(allocator, "default");
    }

    /// Close the store
    pub fn close(self: *Store) void {
        kv_close(self.handle);
    }

    /// Get a value by key
    pub fn get(self: *const Store, key: []const u8) !?core.OwnedBytes {
        return kv_get(self.allocator, self.handle, key);
    }

    /// Get a value as string
    pub fn getString(self: *const Store, key: []const u8) !?core.OwnedString {
        const bytes = try self.get(key) orelse return null;
        return .{
            .data = bytes.data,
            .allocator = bytes.allocator,
        };
    }

    /// Get and parse JSON value
    pub fn getJson(self: *const Store, comptime T: type, key: []const u8) !?T {
        var bytes = try self.get(key) orelse return null;
        defer bytes.deinit();
        return try std.json.parseFromSlice(T, self.allocator, bytes.data, .{});
    }

    /// Set a value
    pub fn set(self: *const Store, key: []const u8, value: []const u8) !void {
        return kv_set(self.handle, key, value);
    }

    /// Set a JSON value
    pub fn setJson(self: *const Store, key: []const u8, value: anytype) !void {
        const json_str = try std.json.stringifyAlloc(self.allocator, value, .{});
        defer self.allocator.free(json_str);
        return self.set(key, json_str);
    }

    /// Delete a key
    pub fn delete(self: *const Store, key: []const u8) !void {
        return kv_delete(self.handle, key);
    }

    /// Check if a key exists
    pub fn exists(self: *const Store, key: []const u8) !bool {
        return kv_exists(self.handle, key);
    }

    /// Get all keys (with optional prefix filter)
    pub fn keys(self: *const Store, prefix: ?[]const u8) !KeyIterator {
        return KeyIterator.init(self.allocator, self.handle, prefix);
    }

    /// Get or set with default
    pub fn getOrSet(
        self: *const Store,
        key: []const u8,
        default_fn: *const fn () []const u8,
    ) !core.OwnedBytes {
        if (try self.get(key)) |value| {
            return value;
        }
        const default = default_fn();
        try self.set(key, default);
        const copy = try self.allocator.dupe(u8, default);
        return .{ .data = copy, .allocator = self.allocator };
    }
};

/// Iterator over store keys
pub const KeyIterator = struct {
    allocator: std.mem.Allocator,
    handle: u32,
    prefix: ?[]const u8,
    cursor: u32,
    done: bool,

    fn init(allocator: std.mem.Allocator, store_handle: u32, prefix: ?[]const u8) !KeyIterator {
        return .{
            .allocator = allocator,
            .handle = store_handle,
            .prefix = prefix,
            .cursor = 0,
            .done = false,
        };
    }

    pub fn next(self: *KeyIterator) !?core.OwnedString {
        if (self.done) return null;

        const result = try kv_keys_next(self.allocator, self.handle, self.prefix, &self.cursor);
        if (result == null) {
            self.done = true;
        }
        return result;
    }

    /// Collect all keys into a slice
    pub fn collect(self: *KeyIterator) ![]core.OwnedString {
        var list = std.ArrayList(core.OwnedString).init(self.allocator);
        while (try self.next()) |key| {
            try list.append(key);
        }
        return list.toOwnedSlice();
    }
};

/// Typed store wrapper for specific value types
pub fn TypedStore(comptime T: type) type {
    return struct {
        store: Store,

        const Self = @This();

        pub fn init(store: Store) Self {
            return .{ .store = store };
        }

        pub fn get(self: *const Self, key: []const u8) !?T {
            return self.store.getJson(T, key);
        }

        pub fn set(self: *const Self, key: []const u8, value: T) !void {
            return self.store.setJson(key, value);
        }

        pub fn delete(self: *const Self, key: []const u8) !void {
            return self.store.delete(key);
        }
    };
}

// Internal: WASM host bindings (to be implemented)
fn kv_open(name: []const u8) !u32 {
    _ = name;
    @panic("kv_open not implemented - requires WASM host bindings");
}

fn kv_close(handle: u32) void {
    _ = handle;
}

fn kv_get(allocator: std.mem.Allocator, handle: u32, key: []const u8) !?core.OwnedBytes {
    _ = allocator;
    _ = handle;
    _ = key;
    @panic("kv_get not implemented - requires WASM host bindings");
}

fn kv_set(handle: u32, key: []const u8, value: []const u8) !void {
    _ = handle;
    _ = key;
    _ = value;
    @panic("kv_set not implemented - requires WASM host bindings");
}

fn kv_delete(handle: u32, key: []const u8) !void {
    _ = handle;
    _ = key;
    @panic("kv_delete not implemented - requires WASM host bindings");
}

fn kv_exists(handle: u32, key: []const u8) !bool {
    _ = handle;
    _ = key;
    @panic("kv_exists not implemented - requires WASM host bindings");
}

fn kv_keys_next(
    allocator: std.mem.Allocator,
    handle: u32,
    prefix: ?[]const u8,
    cursor: *u32,
) !?core.OwnedString {
    _ = allocator;
    _ = handle;
    _ = prefix;
    _ = cursor;
    @panic("kv_keys_next not implemented - requires WASM host bindings");
}
```

### 27.5 Configuration and Variables

```zig
// src/spin/config.zig

const std = @import("std");
const core = @import("core.zig");

/// Configuration access
pub const Config = struct {
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) Config {
        return .{ .allocator = allocator };
    }

    /// Get a configuration value
    pub fn get(self: *const Config, key: []const u8) !?core.OwnedString {
        return config_get(self.allocator, key);
    }

    /// Get a configuration value or default
    pub fn getOr(self: *const Config, key: []const u8, default: []const u8) !core.OwnedString {
        if (try self.get(key)) |value| {
            return value;
        }
        const copy = try self.allocator.dupe(u8, default);
        return .{ .data = copy, .allocator = self.allocator };
    }

    /// Get a required configuration value
    pub fn require(self: *const Config, key: []const u8) !core.OwnedString {
        return try self.get(key) orelse core.SpinError.ConfigNotFound;
    }

    /// Get configuration as integer
    pub fn getInt(self: *const Config, comptime T: type, key: []const u8) !?T {
        var value = try self.get(key) orelse return null;
        defer value.deinit();
        return std.fmt.parseInt(T, value.data, 10) catch return null;
    }

    /// Get configuration as boolean
    pub fn getBool(self: *const Config, key: []const u8) !?bool {
        var value = try self.get(key) orelse return null;
        defer value.deinit();

        const lower = std.ascii.lowerString(value.data, value.data);
        if (std.mem.eql(u8, lower, "true") or std.mem.eql(u8, lower, "1")) {
            return true;
        }
        if (std.mem.eql(u8, lower, "false") or std.mem.eql(u8, lower, "0")) {
            return false;
        }
        return null;
    }
};

/// Spin variables (secrets and configuration)
pub const Variables = struct {
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) Variables {
        return .{ .allocator = allocator };
    }

    /// Get a variable value
    pub fn get(self: *const Variables, name: []const u8) !?core.OwnedString {
        return variables_get(self.allocator, name);
    }

    /// Get a required variable
    pub fn require(self: *const Variables, name: []const u8) !core.OwnedString {
        return try self.get(name) orelse core.SpinError.ConfigNotFound;
    }
};

/// Compile-time checked configuration keys
pub fn TypedConfig(comptime keys: []const []const u8) type {
    return struct {
        config: Config,

        const Self = @This();

        pub fn init(allocator: std.mem.Allocator) Self {
            return .{ .config = Config.init(allocator) };
        }

        pub fn get(self: *const Self, comptime key: []const u8) !?core.OwnedString {
            // Compile-time check that key is valid
            comptime {
                var found = false;
                for (keys) |k| {
                    if (std.mem.eql(u8, k, key)) {
                        found = true;
                        break;
                    }
                }
                if (!found) {
                    @compileError("Invalid config key: " ++ key);
                }
            }
            return self.config.get(key);
        }
    };
}

// Internal: WASM host bindings
fn config_get(allocator: std.mem.Allocator, key: []const u8) !?core.OwnedString {
    _ = allocator;
    _ = key;
    @panic("config_get not implemented - requires WASM host bindings");
}

fn variables_get(allocator: std.mem.Allocator, name: []const u8) !?core.OwnedString {
    _ = allocator;
    _ = name;
    @panic("variables_get not implemented - requires WASM host bindings");
}
```

---

## Phase 2: SQLite

### 27.6 SQLite Database

```zig
// src/spin/sqlite.zig

const std = @import("std");
const core = @import("core.zig");

/// SQLite connection
pub const Connection = struct {
    name: []const u8,
    handle: u32,
    allocator: std.mem.Allocator,

    /// Open a database connection
    pub fn open(allocator: std.mem.Allocator, name: []const u8) !Connection {
        const handle = try sqlite_open(name);
        return .{
            .name = name,
            .handle = handle,
            .allocator = allocator,
        };
    }

    /// Open the default database
    pub fn openDefault(allocator: std.mem.Allocator) !Connection {
        return open(allocator, "default");
    }

    /// Close the connection
    pub fn close(self: *Connection) void {
        sqlite_close(self.handle);
    }

    /// Execute a statement without results
    pub fn execute(self: *const Connection, sql: []const u8, params: anytype) !void {
        var stmt = try self.prepare(sql);
        defer stmt.finalize();
        try stmt.bind(params);
        _ = try stmt.step();
    }

    /// Execute and return affected rows count
    pub fn executeReturningCount(self: *const Connection, sql: []const u8, params: anytype) !usize {
        try self.execute(sql, params);
        return sqlite_changes(self.handle);
    }

    /// Query with results
    pub fn query(self: *const Connection, sql: []const u8, params: anytype) !Rows {
        var stmt = try self.prepare(sql);
        try stmt.bind(params);
        return Rows{ .stmt = stmt, .allocator = self.allocator };
    }

    /// Query a single row
    pub fn queryRow(
        self: *const Connection,
        comptime T: type,
        sql: []const u8,
        params: anytype,
    ) !?T {
        var rows = try self.query(sql, params);
        defer rows.deinit();
        return rows.next(T);
    }

    /// Query all rows
    pub fn queryAll(
        self: *const Connection,
        comptime T: type,
        sql: []const u8,
        params: anytype,
    ) ![]T {
        var rows = try self.query(sql, params);
        defer rows.deinit();
        return rows.collect(T);
    }

    /// Prepare a statement
    pub fn prepare(self: *const Connection, sql: []const u8) !Statement {
        const stmt_handle = try sqlite_prepare(self.handle, sql);
        return Statement{
            .handle = stmt_handle,
            .conn_handle = self.handle,
            .allocator = self.allocator,
        };
    }

    /// Execute in a transaction
    pub fn transaction(self: *const Connection, comptime func: fn (*const Connection) anyerror!void) !void {
        try self.execute("BEGIN TRANSACTION", .{});
        errdefer self.execute("ROLLBACK", .{}) catch {};
        try func(self);
        try self.execute("COMMIT", .{});
    }

    /// Get last insert row ID
    pub fn lastInsertRowId(self: *const Connection) i64 {
        return sqlite_last_insert_rowid(self.handle);
    }
};

/// Prepared statement
pub const Statement = struct {
    handle: u32,
    conn_handle: u32,
    allocator: std.mem.Allocator,

    /// Bind parameters to the statement
    pub fn bind(self: *Statement, params: anytype) !void {
        const T = @TypeOf(params);
        const info = @typeInfo(T);

        if (info == .@"struct" and info.@"struct".is_tuple) {
            inline for (params, 0..) |param, i| {
                try self.bindAt(i + 1, param);
            }
        }
    }

    fn bindAt(self: *Statement, index: usize, value: anytype) !void {
        const T = @TypeOf(value);

        switch (@typeInfo(T)) {
            .int, .comptime_int => try sqlite_bind_int(self.handle, index, @intCast(value)),
            .float, .comptime_float => try sqlite_bind_float(self.handle, index, @floatCast(value)),
            .pointer => |ptr| {
                if (ptr.child == u8) {
                    try sqlite_bind_text(self.handle, index, value);
                }
            },
            .optional => {
                if (value) |v| {
                    try self.bindAt(index, v);
                } else {
                    try sqlite_bind_null(self.handle, index);
                }
            },
            else => @compileError("Unsupported bind type"),
        }
    }

    /// Execute step
    pub fn step(self: *Statement) !bool {
        return sqlite_step(self.handle);
    }

    /// Reset for reuse
    pub fn reset(self: *Statement) !void {
        return sqlite_reset(self.handle);
    }

    /// Finalize and release
    pub fn finalize(self: *Statement) void {
        sqlite_finalize(self.handle);
    }

    /// Get column count
    pub fn columnCount(self: *const Statement) usize {
        return sqlite_column_count(self.handle);
    }

    /// Get column value
    pub fn column(self: *const Statement, comptime T: type, index: usize) !T {
        return switch (@typeInfo(T)) {
            .int => @intCast(sqlite_column_int(self.handle, index)),
            .float => @floatCast(sqlite_column_float(self.handle, index)),
            .pointer => |ptr| blk: {
                if (ptr.child == u8) {
                    break :blk sqlite_column_text(self.allocator, self.handle, index);
                }
                @compileError("Unsupported column type");
            },
            .optional => |opt| blk: {
                if (sqlite_column_is_null(self.handle, index)) {
                    break :blk null;
                }
                break :blk try self.column(opt.child, index);
            },
            else => @compileError("Unsupported column type"),
        };
    }
};

/// Result rows iterator
pub const Rows = struct {
    stmt: Statement,
    allocator: std.mem.Allocator,

    pub fn deinit(self: *Rows) void {
        self.stmt.finalize();
    }

    /// Get next row as struct
    pub fn next(self: *Rows, comptime T: type) !?T {
        if (!try self.stmt.step()) {
            return null;
        }

        const info = @typeInfo(T).@"struct";
        var result: T = undefined;

        inline for (info.fields, 0..) |field, i| {
            @field(result, field.name) = try self.stmt.column(field.type, i);
        }

        return result;
    }

    /// Collect all rows
    pub fn collect(self: *Rows, comptime T: type) ![]T {
        var list = std.ArrayList(T).init(self.allocator);
        while (try self.next(T)) |row| {
            try list.append(row);
        }
        return list.toOwnedSlice();
    }
};

/// Query builder for type-safe queries
pub fn QueryBuilder(comptime T: type) type {
    return struct {
        conn: *const Connection,
        table: []const u8,
        where_clause: ?[]const u8,
        order_by: ?[]const u8,
        limit_value: ?usize,

        const Self = @This();

        pub fn init(conn: *const Connection, table: []const u8) Self {
            return .{
                .conn = conn,
                .table = table,
                .where_clause = null,
                .order_by = null,
                .limit_value = null,
            };
        }

        pub fn where(self: *Self, clause: []const u8) *Self {
            self.where_clause = clause;
            return self;
        }

        pub fn orderBy(self: *Self, column: []const u8) *Self {
            self.order_by = column;
            return self;
        }

        pub fn limit(self: *Self, n: usize) *Self {
            self.limit_value = n;
            return self;
        }

        pub fn all(self: *Self, params: anytype) ![]T {
            const sql = self.buildSelect();
            return self.conn.queryAll(T, sql, params);
        }

        pub fn first(self: *Self, params: anytype) !?T {
            self.limit_value = 1;
            const sql = self.buildSelect();
            return self.conn.queryRow(T, sql, params);
        }

        fn buildSelect(self: *const Self) []const u8 {
            // Build SELECT query from fields of T
            _ = self;
            @panic("QueryBuilder.buildSelect not implemented");
        }
    };
}

// Internal: WASM host bindings
fn sqlite_open(name: []const u8) !u32 {
    _ = name;
    @panic("sqlite_open not implemented");
}

fn sqlite_close(handle: u32) void {
    _ = handle;
}

fn sqlite_prepare(conn: u32, sql: []const u8) !u32 {
    _ = conn;
    _ = sql;
    @panic("sqlite_prepare not implemented");
}

fn sqlite_step(stmt: u32) !bool {
    _ = stmt;
    @panic("sqlite_step not implemented");
}

fn sqlite_reset(stmt: u32) !void {
    _ = stmt;
    @panic("sqlite_reset not implemented");
}

fn sqlite_finalize(stmt: u32) void {
    _ = stmt;
}

fn sqlite_changes(conn: u32) usize {
    _ = conn;
    @panic("sqlite_changes not implemented");
}

fn sqlite_last_insert_rowid(conn: u32) i64 {
    _ = conn;
    @panic("sqlite_last_insert_rowid not implemented");
}

fn sqlite_column_count(stmt: u32) usize {
    _ = stmt;
    @panic("sqlite_column_count not implemented");
}

fn sqlite_column_int(stmt: u32, index: usize) i64 {
    _ = stmt;
    _ = index;
    @panic("sqlite_column_int not implemented");
}

fn sqlite_column_float(stmt: u32, index: usize) f64 {
    _ = stmt;
    _ = index;
    @panic("sqlite_column_float not implemented");
}

fn sqlite_column_text(allocator: std.mem.Allocator, stmt: u32, index: usize) ![]const u8 {
    _ = allocator;
    _ = stmt;
    _ = index;
    @panic("sqlite_column_text not implemented");
}

fn sqlite_column_is_null(stmt: u32, index: usize) bool {
    _ = stmt;
    _ = index;
    @panic("sqlite_column_is_null not implemented");
}

fn sqlite_bind_int(stmt: u32, index: usize, value: i64) !void {
    _ = stmt;
    _ = index;
    _ = value;
    @panic("sqlite_bind_int not implemented");
}

fn sqlite_bind_float(stmt: u32, index: usize, value: f64) !void {
    _ = stmt;
    _ = index;
    _ = value;
    @panic("sqlite_bind_float not implemented");
}

fn sqlite_bind_text(stmt: u32, index: usize, value: []const u8) !void {
    _ = stmt;
    _ = index;
    _ = value;
    @panic("sqlite_bind_text not implemented");
}

fn sqlite_bind_null(stmt: u32, index: usize) !void {
    _ = stmt;
    _ = index;
    @panic("sqlite_bind_null not implemented");
}
```

---

## Phase 3: Future Modules

### 27.7 Redis (Interface Sketch)

```zig
// src/spin/future/redis.zig

const std = @import("std");
const core = @import("../core.zig");

pub const Redis = struct {
    address: []const u8,
    allocator: std.mem.Allocator,

    pub fn connect(allocator: std.mem.Allocator, address: []const u8) !Redis {
        _ = allocator;
        _ = address;
        @panic("Redis not yet implemented");
    }

    // String operations
    pub fn get(self: *Redis, key: []const u8) !?core.OwnedBytes { _ = self; _ = key; @panic("not implemented"); }
    pub fn set(self: *Redis, key: []const u8, value: []const u8) !void { _ = self; _ = key; _ = value; @panic("not implemented"); }
    pub fn setex(self: *Redis, key: []const u8, value: []const u8, ttl_seconds: u32) !void { _ = self; _ = key; _ = value; _ = ttl_seconds; @panic("not implemented"); }
    pub fn del(self: *Redis, key: []const u8) !void { _ = self; _ = key; @panic("not implemented"); }
    pub fn incr(self: *Redis, key: []const u8) !i64 { _ = self; _ = key; @panic("not implemented"); }
    pub fn incrby(self: *Redis, key: []const u8, amount: i64) !i64 { _ = self; _ = key; _ = amount; @panic("not implemented"); }

    // Set operations
    pub fn sadd(self: *Redis, key: []const u8, values: []const []const u8) !usize { _ = self; _ = key; _ = values; @panic("not implemented"); }
    pub fn smembers(self: *Redis, key: []const u8) ![]core.OwnedBytes { _ = self; _ = key; @panic("not implemented"); }
    pub fn srem(self: *Redis, key: []const u8, values: []const []const u8) !usize { _ = self; _ = key; _ = values; @panic("not implemented"); }

    // Pub/Sub
    pub fn publish(self: *Redis, channel: []const u8, message: []const u8) !void { _ = self; _ = channel; _ = message; @panic("not implemented"); }
};
```

### 27.8 PostgreSQL (Interface Sketch)

```zig
// src/spin/future/postgres.zig

const std = @import("std");
const core = @import("../core.zig");

pub const Postgres = struct {
    address: []const u8,
    allocator: std.mem.Allocator,

    pub fn connect(allocator: std.mem.Allocator, address: []const u8) !Postgres {
        _ = allocator;
        _ = address;
        @panic("PostgreSQL not yet implemented");
    }

    pub fn execute(self: *Postgres, sql: []const u8, params: anytype) !void { _ = self; _ = sql; _ = params; @panic("not implemented"); }
    pub fn query(self: *Postgres, comptime T: type, sql: []const u8, params: anytype) ![]T { _ = self; _ = sql; _ = params; @panic("not implemented"); }
    pub fn queryRow(self: *Postgres, comptime T: type, sql: []const u8, params: anytype) !?T { _ = self; _ = sql; _ = params; @panic("not implemented"); }
};
```

### 27.9 LLM (Interface Sketch)

```zig
// src/spin/future/llm.zig

const std = @import("std");
const core = @import("../core.zig");

pub const Model = enum {
    llama2_chat,
    codellama_instruct,
    // Add more models as Spin supports them
};

pub const InferenceOptions = struct {
    max_tokens: u32 = 100,
    temperature: f32 = 0.8,
    top_p: f32 = 0.9,
    repeat_penalty: f32 = 1.1,
};

pub const LLM = struct {
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) LLM {
        return .{ .allocator = allocator };
    }

    pub fn infer(
        self: *LLM,
        model: Model,
        prompt: []const u8,
        options: InferenceOptions,
    ) !core.OwnedString {
        _ = self;
        _ = model;
        _ = prompt;
        _ = options;
        @panic("LLM inference not yet implemented");
    }

    pub fn generateEmbeddings(
        self: *LLM,
        model: Model,
        texts: []const []const u8,
    ) ![][]f32 {
        _ = self;
        _ = model;
        _ = texts;
        @panic("LLM embeddings not yet implemented");
    }
};
```

---

## WIT Bindings Integration

### Implementation Strategy

The SDK bridges to Spin's host functions through WIT (WebAssembly Interface Types). Since `wit-bindgen` generates C code, we use `@cImport`:

```zig
// src/spin/bindings.zig

const c = @cImport({
    @cInclude("spin-http.h");
    @cInclude("spin-kv.h");
    @cInclude("spin-config.h");
    @cInclude("spin-sqlite.h");
});

/// Convert C types to Zig types
pub const Bindings = struct {
    // HTTP
    pub fn httpSend(req: *const c.spin_http_request_t) !c.spin_http_response_t {
        var response: c.spin_http_response_t = undefined;
        const result = c.spin_http_send(req, &response);
        if (result != 0) return core.SpinError.HttpRequestFailed;
        return response;
    }

    // KV Store
    pub fn kvOpen(name: []const u8) !u32 {
        var handle: u32 = undefined;
        const result = c.spin_kv_open(name.ptr, name.len, &handle);
        if (result != 0) return core.SpinError.StoreNotFound;
        return handle;
    }

    // ... more bindings
};
```

### Build Configuration

```zig
// build.zig additions for Spin SDK

const spin_sdk = b.addModule("spin-sdk", .{
    .root_source_file = b.path("src/spin/root.zig"),
});

// Link C bindings generated by wit-bindgen
spin_sdk.linkLibC();
spin_sdk.addIncludePath(b.path("wit-generated/"));
spin_sdk.addCSourceFile(b.path("wit-generated/spin-bindings.c"), &.{});
```

---

## Integration with Zuif

### Server Module Usage

```zig
// Example: zuif server using Spin SDK
const std = @import("std");
const spin = @import("spin-sdk");
const zuif = @import("zuif");

pub fn handleRequest(req: *spin.http.Request, allocator: std.mem.Allocator) !spin.http.Response {
    // Route handling
    const router = zuif.Router.init();

    // Use KV store for session
    var store = try spin.kv.Store.openDefault(allocator);
    defer store.close();

    // Get config
    const config = spin.config.Config.init(allocator);
    const api_key = try config.get("api_key");

    // Render component
    const html = try zuif.render(MyComponent, .{ .user = user }, allocator);

    var response = spin.http.Response.init(allocator);
    return try response.html(html);
}
```

### Effect Integration

```zig
// HTTP effect using Spin SDK
pub fn createHttpEffect(comptime T: type) type {
    return struct {
        pub fn execute(
            url: []const u8,
            options: HttpOptions,
            allocator: std.mem.Allocator,
        ) !T {
            var client = spin.http.Client.init(allocator);
            defer client.deinit();

            var req = try client.request(url);
            defer req.deinit();

            _ = req.setMethod(options.method);
            if (options.body) |body| {
                _ = try req.json(body);
            }

            var resp = try req.send();
            defer resp.deinit();

            return try resp.json(T);
        }
    };
}
```

---

## Testing Strategy

### Mock Host Functions

```zig
// src/spin/testing.zig

const std = @import("std");

/// Mock KV store for testing
pub const MockKvStore = struct {
    data: std.StringHashMap([]const u8),
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) MockKvStore {
        return .{
            .data = std.StringHashMap([]const u8).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn get(self: *MockKvStore, key: []const u8) ?[]const u8 {
        return self.data.get(key);
    }

    pub fn set(self: *MockKvStore, key: []const u8, value: []const u8) !void {
        const k = try self.allocator.dupe(u8, key);
        const v = try self.allocator.dupe(u8, value);
        try self.data.put(k, v);
    }
};

/// Mock HTTP client for testing
pub const MockHttpClient = struct {
    responses: std.ArrayList(MockResponse),

    pub const MockResponse = struct {
        url_pattern: []const u8,
        status: u16,
        body: []const u8,
    };

    pub fn expectGet(self: *MockHttpClient, url: []const u8, response: []const u8) !void {
        try self.responses.append(.{
            .url_pattern = url,
            .status = 200,
            .body = response,
        });
    }
};
```

### Unit Tests

```zig
test "KV store get/set" {
    var store = testing.MockKvStore.init(std.testing.allocator);
    defer store.deinit();

    try store.set("key", "value");
    const value = store.get("key");

    try std.testing.expectEqualStrings("value", value.?);
}

test "HTTP client request building" {
    var client = spin.http.Client.init(std.testing.allocator);
    defer client.deinit();

    _ = client.setBaseUrl("https://api.example.com");
    _ = try client.setDefaultHeader("Authorization", "Bearer token");

    var req = try client.request("/users");
    defer req.deinit();

    try std.testing.expectEqualStrings("https://api.example.com/users", req.url);
}
```

---

## Error Handling Patterns

### Unified Error Type

```zig
/// Map host errors to SpinError
fn mapHostError(code: i32) core.SpinError {
    return switch (code) {
        1 => core.SpinError.StoreNotFound,
        2 => core.SpinError.KeyNotFound,
        3 => core.SpinError.PermissionDenied,
        4 => core.SpinError.ResourceExhausted,
        else => core.SpinError.InternalError,
    };
}
```

### Error Context

```zig
/// Error with context information
pub const ErrorContext = struct {
    err: core.SpinError,
    message: []const u8,
    source: ?[]const u8,

    pub fn format(
        self: ErrorContext,
        comptime fmt: []const u8,
        options: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        _ = fmt;
        _ = options;
        try writer.print("SpinError.{s}: {s}", .{
            @tagName(self.err),
            self.message,
        });
        if (self.source) |src| {
            try writer.print(" (in {s})", .{src});
        }
    }
};
```

---

## Implementation Notes

### Memory Safety

1. **Allocator Awareness**: All APIs accept allocators for memory management
2. **Owned Types**: `OwnedBytes` and `OwnedString` track ownership
3. **Defer Cleanup**: APIs designed for easy `defer deinit()` patterns
4. **No Hidden Allocations**: All allocations are explicit

### WASM Considerations

1. **Linear Memory**: Single memory space, careful buffer management
2. **No Threads**: Zig's async not available in WASM
3. **Stack Size**: Limited stack, prefer heap for large data
4. **Host Calls**: Minimize round-trips to host functions

### Performance

1. **Zero-Copy Where Possible**: Return slices into host memory when safe
2. **Batch Operations**: Provide batch APIs for KV and database
3. **Connection Reuse**: Pool handles when appropriate
4. **Lazy Parsing**: Parse JSON/data only when accessed

---

## Specification Status

| Module | Status | Priority |
|--------|--------|----------|
| Core Types | Specified | Critical |
| HTTP Inbound | Specified | Critical |
| HTTP Outbound | Specified | Critical |
| Key-Value | Specified | Critical |
| Config | Specified | Critical |
| SQLite | Specified | High |
| Redis | Interface Only | Future |
| PostgreSQL | Interface Only | Future |
| MySQL | Interface Only | Future |
| LLM | Interface Only | Future |

## Related Specifications

- [03-components.md](03-components.md) - Component architecture
- [11-effects.md](11-effects.md) - Effect system integration
- [12-server-client.md](12-server-client.md) - Server/client model
- [22-suspense.md](22-suspense.md) - Async data loading
