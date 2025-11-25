# 20 - BROWSER APIS

**Document:** Browser API Integration
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

ZUI provides type-safe access to browser APIs from Zig/WASM. This document specifies the available APIs, their Zig interfaces, and the underlying JS bridge implementation.

---

## ARCHITECTURE

### The Bridge Pattern

WASM cannot directly access browser APIs. All calls go through JavaScript:

```
┌─────────────────────────────────────────────────────────────┐
│                        ZIG / WASM                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │   const uuid = try browser.crypto.randomUUID();     │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                            ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │   extern "zui" fn js_crypto_random_uuid() ...       │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
├────────────────────────────┼────────────────────────────────┤
│                     WASM BOUNDARY                           │
├────────────────────────────┼────────────────────────────────┤
│                            │                                │
│  ┌─────────────────────────▼───────────────────────────┐   │
│  │   js_crypto_random_uuid() {                         │   │
│  │       return crypto.randomUUID();                   │   │
│  │   }                                                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                            ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Browser APIs (native)                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│                      JAVASCRIPT                             │
└─────────────────────────────────────────────────────────────┘
```

### Module Structure

```
src/browser/
├── mod.zig              // Re-exports all modules
├── storage.zig          // localStorage, sessionStorage
├── indexed_db.zig       // IndexedDB (async)
├── crypto.zig           // Web Crypto API
├── clipboard.zig        // Clipboard API
├── notifications.zig    // Notifications API
├── geolocation.zig      // Geolocation API
├── websocket.zig        // WebSocket connections
├── workers.zig          // Web Workers
├── intl.zig             // Internationalization
├── media.zig            // Media queries, fullscreen
├── share.zig            // Web Share API
├── performance.zig      // Performance API
└── broadcast.zig        // Broadcast Channel (multi-tab)
```

### Usage Pattern

```zig
const zui = @import("zui");
const browser = zui.browser;

// Direct API access (synchronous)
browser.storage.local.set("key", "value");
const value = browser.storage.local.get(allocator, "key");

// Effect-based access (asynchronous)
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .copy_text => browser.clipboard.write(model.text, .{
            .on_success = .text_copied,
            .on_error = .copy_failed,
        }),
    };
}
```

---

## STORAGE APIs

### LocalStorage & SessionStorage

```zig
// src/browser/storage.zig
const std = @import("std");

pub const Storage = struct {
    store_type: StoreType,

    pub const StoreType = enum(u8) {
        local = 0,
        session = 1,
    };

    /// Get a value from storage
    pub fn get(self: Storage, allocator: std.mem.Allocator, key: []const u8) !?[]u8 {
        const result_len = js_storage_get(
            @intFromEnum(self.store_type),
            key.ptr,
            key.len,
        );

        if (result_len == 0) return null;

        const buffer = try allocator.alloc(u8, result_len);
        _ = js_storage_read(buffer.ptr, result_len);
        return buffer;
    }

    /// Get and parse JSON value
    pub fn getJson(self: Storage, comptime T: type, allocator: std.mem.Allocator, key: []const u8) !?T {
        const json_str = try self.get(allocator, key) orelse return null;
        defer allocator.free(json_str);

        return try std.json.parseFromSlice(T, allocator, json_str, .{});
    }

    /// Set a value in storage
    pub fn set(self: Storage, key: []const u8, value: []const u8) void {
        js_storage_set(
            @intFromEnum(self.store_type),
            key.ptr, key.len,
            value.ptr, value.len,
        );
    }

    /// Set a JSON-serializable value
    pub fn setJson(self: Storage, key: []const u8, value: anytype, allocator: std.mem.Allocator) !void {
        const json_str = try std.json.stringifyAlloc(allocator, value, .{});
        defer allocator.free(json_str);
        self.set(key, json_str);
    }

    /// Remove a key from storage
    pub fn remove(self: Storage, key: []const u8) void {
        js_storage_remove(@intFromEnum(self.store_type), key.ptr, key.len);
    }

    /// Clear all storage
    pub fn clear(self: Storage) void {
        js_storage_clear(@intFromEnum(self.store_type));
    }

    /// Get number of stored items
    pub fn length(self: Storage) usize {
        return js_storage_length(@intFromEnum(self.store_type));
    }

    /// Get key at index
    pub fn key(self: Storage, allocator: std.mem.Allocator, index: usize) !?[]u8 {
        const result_len = js_storage_key(@intFromEnum(self.store_type), index);
        if (result_len == 0) return null;

        const buffer = try allocator.alloc(u8, result_len);
        _ = js_storage_read(buffer.ptr, result_len);
        return buffer;
    }
};

/// localStorage - persists across browser sessions
pub const local = Storage{ .store_type = .local };

/// sessionStorage - cleared when tab closes
pub const session = Storage{ .store_type = .session };

// JS Imports
extern "zui" fn js_storage_get(store: u8, key_ptr: [*]const u8, key_len: usize) usize;
extern "zui" fn js_storage_read(buf_ptr: [*]u8, buf_len: usize) usize;
extern "zui" fn js_storage_set(store: u8, key_ptr: [*]const u8, key_len: usize, val_ptr: [*]const u8, val_len: usize) void;
extern "zui" fn js_storage_remove(store: u8, key_ptr: [*]const u8, key_len: usize) void;
extern "zui" fn js_storage_clear(store: u8) void;
extern "zui" fn js_storage_length(store: u8) usize;
extern "zui" fn js_storage_key(store: u8, index: usize) usize;
```

### IndexedDB (Async)

For larger data and structured storage:

```zig
// src/browser/indexed_db.zig
pub const IndexedDB = struct {
    /// Open a database (async - returns Effect)
    pub fn open(name: []const u8, version: u32) Effect(DatabaseHandle) {
        return Effect.browser(.{
            .operation = .idb_open,
            .name = name,
            .version = version,
        });
    }

    /// Database operations
    pub const Database = struct {
        handle: u32,

        pub fn createObjectStore(self: Database, name: []const u8, options: ObjectStoreOptions) void {
            js_idb_create_store(self.handle, name.ptr, name.len, options);
        }

        pub fn transaction(self: Database, stores: []const []const u8, mode: TransactionMode) Transaction {
            // ...
        }
    };

    pub const Transaction = struct {
        handle: u32,

        pub fn objectStore(self: Transaction, name: []const u8) ObjectStore {
            // ...
        }
    };

    pub const ObjectStore = struct {
        handle: u32,

        pub fn put(self: ObjectStore, key: []const u8, value: []const u8) Effect(void) {
            return Effect.browser(.{ .operation = .idb_put, .store = self.handle, .key = key, .value = value });
        }

        pub fn get(self: ObjectStore, key: []const u8) Effect(?[]const u8) {
            return Effect.browser(.{ .operation = .idb_get, .store = self.handle, .key = key });
        }

        pub fn delete(self: ObjectStore, key: []const u8) Effect(void) {
            return Effect.browser(.{ .operation = .idb_delete, .store = self.handle, .key = key });
        }
    };
};
```

---

## CRYPTO API

### Random Values

```zig
// src/browser/crypto.zig
pub const Crypto = struct {
    /// Fill buffer with cryptographically secure random bytes
    pub fn getRandomValues(buffer: []u8) void {
        js_crypto_get_random_values(buffer.ptr, buffer.len);
    }

    /// Generate random bytes and return owned slice
    pub fn randomBytes(allocator: std.mem.Allocator, len: usize) ![]u8 {
        const buffer = try allocator.alloc(u8, len);
        getRandomValues(buffer);
        return buffer;
    }

    /// Generate a random UUID v4
    pub fn randomUUID() [36]u8 {
        var buffer: [36]u8 = undefined;
        js_crypto_random_uuid(&buffer);
        return buffer;
    }

    /// Generate random integer in range [0, max)
    pub fn randomInt(max: u32) u32 {
        var bytes: [4]u8 = undefined;
        getRandomValues(&bytes);
        const value = std.mem.readInt(u32, &bytes, .little);
        return value % max;
    }
};
```

### Hashing

```zig
pub const Hash = struct {
    pub const Algorithm = enum {
        sha1,    // Legacy, not recommended
        sha256,
        sha384,
        sha512,
    };

    /// Hash data synchronously (small data only)
    pub fn digest(algorithm: Algorithm, data: []const u8) Effect([]const u8) {
        return Effect.browser(.{
            .operation = .crypto_hash,
            .algorithm = algorithm,
            .data = data,
        });
    }

    /// Compute hash and return hex string
    pub fn digestHex(algorithm: Algorithm, data: []const u8) Effect([]const u8) {
        return Effect.browser(.{
            .operation = .crypto_hash_hex,
            .algorithm = algorithm,
            .data = data,
        });
    }
};
```

### Encryption (AES-GCM)

```zig
pub const Encryption = struct {
    pub const Key = struct {
        handle: u32,

        /// Generate a new random key
        pub fn generate(bits: u16) Effect(Key) {
            return Effect.browser(.{
                .operation = .crypto_generate_key,
                .bits = bits, // 128, 192, or 256
            });
        }

        /// Import key from raw bytes
        pub fn import(key_data: []const u8) Effect(Key) {
            return Effect.browser(.{
                .operation = .crypto_import_key,
                .data = key_data,
            });
        }

        /// Export key to raw bytes
        pub fn export(self: Key) Effect([]const u8) {
            return Effect.browser(.{
                .operation = .crypto_export_key,
                .key = self.handle,
            });
        }
    };

    /// Encrypt data with AES-GCM
    pub fn encrypt(key: Key, plaintext: []const u8) Effect(EncryptedData) {
        return Effect.browser(.{
            .operation = .crypto_encrypt,
            .key = key.handle,
            .data = plaintext,
        });
    }

    /// Decrypt data with AES-GCM
    pub fn decrypt(key: Key, encrypted: EncryptedData) Effect([]const u8) {
        return Effect.browser(.{
            .operation = .crypto_decrypt,
            .key = key.handle,
            .iv = encrypted.iv,
            .data = encrypted.ciphertext,
            .tag = encrypted.tag,
        });
    }

    pub const EncryptedData = struct {
        iv: [12]u8,
        ciphertext: []const u8,
        tag: [16]u8,
    };
};
```

### HMAC & PBKDF2

```zig
pub const HMAC = struct {
    /// Sign data with HMAC
    pub fn sign(algorithm: Hash.Algorithm, key: []const u8, data: []const u8) Effect([]const u8) {
        return Effect.browser(.{
            .operation = .crypto_hmac_sign,
            .algorithm = algorithm,
            .key = key,
            .data = data,
        });
    }

    /// Verify HMAC signature
    pub fn verify(algorithm: Hash.Algorithm, key: []const u8, data: []const u8, signature: []const u8) Effect(bool) {
        return Effect.browser(.{
            .operation = .crypto_hmac_verify,
            .algorithm = algorithm,
            .key = key,
            .data = data,
            .signature = signature,
        });
    }
};

pub const PBKDF2 = struct {
    /// Derive key from password
    pub fn deriveKey(
        password: []const u8,
        salt: []const u8,
        iterations: u32,
        key_length: u32,
    ) Effect([]const u8) {
        return Effect.browser(.{
            .operation = .crypto_pbkdf2,
            .password = password,
            .salt = salt,
            .iterations = iterations,
            .key_length = key_length,
        });
    }
};
```

---

## CLIPBOARD API

```zig
// src/browser/clipboard.zig
pub const Clipboard = struct {
    /// Write text to clipboard (async, requires user gesture)
    pub fn writeText(text: []const u8) Effect(void) {
        return Effect.browser(.{
            .operation = .clipboard_write_text,
            .data = text,
        });
    }

    /// Read text from clipboard (async, requires permission)
    pub fn readText() Effect([]const u8) {
        return Effect.browser(.{
            .operation = .clipboard_read_text,
        });
    }

    /// Check if clipboard API is available
    pub fn isAvailable() bool {
        return js_clipboard_available() != 0;
    }
};

// Effect integration example
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .copy_clicked => Clipboard.writeText(model.text_to_copy).map(.copy_success, .copy_failed),
        .paste_clicked => Clipboard.readText().map(.paste_received, .paste_failed),
        .paste_received => |text| {
            model.pasted_text = text;
            return .none;
        },
        // ...
    };
}
```

---

## NOTIFICATIONS API

```zig
// src/browser/notifications.zig
pub const Notifications = struct {
    pub const Permission = enum {
        default,   // Not yet asked
        granted,
        denied,
    };

    /// Get current permission status
    pub fn permission() Permission {
        return @enumFromInt(js_notification_permission());
    }

    /// Request permission from user (async)
    pub fn requestPermission() Effect(Permission) {
        return Effect.browser(.{
            .operation = .notification_request_permission,
        });
    }

    /// Show a notification
    pub fn show(title: []const u8, options: NotificationOptions) Effect(void) {
        return Effect.browser(.{
            .operation = .notification_show,
            .title = title,
            .body = options.body,
            .icon = options.icon,
            .tag = options.tag,
        });
    }

    pub const NotificationOptions = struct {
        body: ?[]const u8 = null,
        icon: ?[]const u8 = null,
        tag: ?[]const u8 = null,  // Replace notification with same tag
        silent: bool = false,
    };
};
```

---

## WEBSOCKET API

```zig
// src/browser/websocket.zig
pub const WebSocket = struct {
    handle: u32,

    pub const State = enum {
        connecting,
        open,
        closing,
        closed,
    };

    /// Connect to WebSocket server
    pub fn connect(url: []const u8) Effect(WebSocket) {
        return Effect.browser(.{
            .operation = .websocket_connect,
            .url = url,
        });
    }

    /// Send text message
    pub fn send(self: WebSocket, data: []const u8) !void {
        if (js_websocket_send(self.handle, data.ptr, data.len) == 0) {
            return error.WebSocketNotOpen;
        }
    }

    /// Send binary data
    pub fn sendBinary(self: WebSocket, data: []const u8) !void {
        if (js_websocket_send_binary(self.handle, data.ptr, data.len) == 0) {
            return error.WebSocketNotOpen;
        }
    }

    /// Close connection
    pub fn close(self: WebSocket) void {
        js_websocket_close(self.handle);
    }

    /// Get current state
    pub fn state(self: WebSocket) State {
        return @enumFromInt(js_websocket_state(self.handle));
    }

    /// Subscribe to events (used internally by Effect system)
    pub const Events = struct {
        on_open: ?Msg,
        on_message: ?fn ([]const u8) Msg,
        on_close: ?Msg,
        on_error: ?Msg,
    };
};

// Usage in application
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .connect_ws => WebSocket.connect("wss://api.example.com/ws").withEvents(.{
            .on_open = .ws_connected,
            .on_message = parseWsMessage,
            .on_close = .ws_disconnected,
            .on_error = .ws_error,
        }),
        .ws_connected => |ws| {
            model.socket = ws;
            return ws.send("{\"type\":\"subscribe\"}");
        },
        .send_message => {
            if (model.socket) |ws| {
                try ws.send(model.message_to_send);
            }
            return .none;
        },
        // ...
    };
}
```

---

## BROADCAST CHANNEL (Multi-Tab Sync)

```zig
// src/browser/broadcast.zig
pub const BroadcastChannel = struct {
    handle: u32,

    /// Create or join a channel
    pub fn init(name: []const u8) BroadcastChannel {
        return .{ .handle = js_broadcast_channel_create(name.ptr, name.len) };
    }

    /// Post message to all other tabs
    pub fn postMessage(self: BroadcastChannel, data: []const u8) void {
        js_broadcast_channel_post(self.handle, data.ptr, data.len);
    }

    /// Close channel
    pub fn close(self: BroadcastChannel) void {
        js_broadcast_channel_close(self.handle);
    }

    /// Subscribe to messages (returns Effect subscription)
    pub fn subscribe(self: BroadcastChannel, comptime handler: fn ([]const u8) Msg) Effect.Subscription {
        return .{
            .type = .broadcast_channel,
            .handle = self.handle,
            .handler = handler,
        };
    }
};

// Usage: Sync state across tabs
pub fn subscriptions(model: *const Model) []const Subscription {
    return &.{
        model.sync_channel.subscribe(handleSyncMessage),
    };
}

fn handleSyncMessage(data: []const u8) Msg {
    // Parse sync message and return appropriate Msg
    const sync = std.json.parse(SyncMessage, data, .{}) catch return .sync_error;
    return .{ .state_synced = sync };
}
```

---

## GEOLOCATION API

```zig
// src/browser/geolocation.zig
pub const Geolocation = struct {
    /// Get current position (one-time)
    pub fn getCurrentPosition(options: PositionOptions) Effect(Position) {
        return Effect.browser(.{
            .operation = .geolocation_get,
            .high_accuracy = options.high_accuracy,
            .timeout = options.timeout,
            .max_age = options.max_age,
        });
    }

    /// Watch position changes (subscription)
    pub fn watchPosition(options: PositionOptions, comptime handler: fn (Position) Msg) Effect.Subscription {
        return .{
            .type = .geolocation_watch,
            .options = options,
            .handler = handler,
        };
    }

    pub const Position = struct {
        latitude: f64,
        longitude: f64,
        accuracy: f64,           // meters
        altitude: ?f64,
        altitude_accuracy: ?f64,
        heading: ?f64,           // degrees from north
        speed: ?f64,             // meters per second
        timestamp: u64,
    };

    pub const PositionOptions = struct {
        high_accuracy: bool = false,
        timeout: u32 = 10000,    // milliseconds
        max_age: u32 = 0,        // milliseconds
    };
};
```

---

## INTERNATIONALIZATION (Intl)

```zig
// src/browser/intl.zig
pub const Intl = struct {
    /// Format a number according to locale
    pub fn formatNumber(
        allocator: std.mem.Allocator,
        value: f64,
        locale: []const u8,
        options: NumberFormatOptions,
    ) ![]u8 {
        const options_json = try std.json.stringifyAlloc(allocator, options, .{});
        defer allocator.free(options_json);

        const result_len = js_intl_format_number(
            value,
            locale.ptr, locale.len,
            options_json.ptr, options_json.len,
        );

        const buffer = try allocator.alloc(u8, result_len);
        _ = js_intl_read(buffer.ptr, result_len);
        return buffer;
    }

    pub const NumberFormatOptions = struct {
        style: enum { decimal, currency, percent, unit } = .decimal,
        currency: ?[]const u8 = null,          // e.g., "USD", "EUR"
        currency_display: ?[]const u8 = null,  // "symbol", "code", "name"
        minimum_fraction_digits: ?u8 = null,
        maximum_fraction_digits: ?u8 = null,
        use_grouping: bool = true,
    };

    /// Format a date/time according to locale
    pub fn formatDateTime(
        allocator: std.mem.Allocator,
        timestamp_ms: i64,
        locale: []const u8,
        options: DateTimeFormatOptions,
    ) ![]u8 {
        // Similar implementation
    }

    pub const DateTimeFormatOptions = struct {
        date_style: ?enum { full, long, medium, short } = null,
        time_style: ?enum { full, long, medium, short } = null,
        weekday: ?enum { long, short, narrow } = null,
        year: ?enum { numeric, two_digit } = null,
        month: ?enum { numeric, two_digit, long, short, narrow } = null,
        day: ?enum { numeric, two_digit } = null,
        hour: ?enum { numeric, two_digit } = null,
        minute: ?enum { numeric, two_digit } = null,
        second: ?enum { numeric, two_digit } = null,
        time_zone: ?[]const u8 = null,
    };

    /// Format relative time (e.g., "3 days ago")
    pub fn formatRelativeTime(
        allocator: std.mem.Allocator,
        value: i64,
        unit: RelativeTimeUnit,
        locale: []const u8,
    ) ![]u8 {
        // ...
    }

    pub const RelativeTimeUnit = enum {
        second, minute, hour, day, week, month, quarter, year,
    };

    /// Get plural category for a number
    pub fn pluralRules(locale: []const u8, value: f64) PluralCategory {
        return @enumFromInt(js_intl_plural_rules(locale.ptr, locale.len, value));
    }

    pub const PluralCategory = enum { zero, one, two, few, many, other };
};
```

---

## MEDIA QUERIES

```zig
// src/browser/media.zig
pub const Media = struct {
    /// Check if media query matches
    pub fn matches(query: []const u8) bool {
        return js_media_query_matches(query.ptr, query.len) != 0;
    }

    /// Common queries
    pub fn prefersDarkMode() bool {
        return matches("(prefers-color-scheme: dark)");
    }

    pub fn prefersReducedMotion() bool {
        return matches("(prefers-reduced-motion: reduce)");
    }

    pub fn isMobile() bool {
        return matches("(max-width: 768px)");
    }

    pub fn isTablet() bool {
        return matches("(min-width: 769px) and (max-width: 1024px)");
    }

    pub fn isDesktop() bool {
        return matches("(min-width: 1025px)");
    }

    /// Watch for media query changes (subscription)
    pub fn watch(query: []const u8, comptime handler: fn (bool) Msg) Effect.Subscription {
        return .{
            .type = .media_query_watch,
            .query = query,
            .handler = handler,
        };
    }
};

/// Fullscreen API
pub const Fullscreen = struct {
    pub fn request(element_id: ?[]const u8) Effect(void) {
        return Effect.browser(.{
            .operation = .fullscreen_request,
            .element = element_id,
        });
    }

    pub fn exit() void {
        js_fullscreen_exit();
    }

    pub fn isActive() bool {
        return js_fullscreen_is_active() != 0;
    }
};
```

---

## WEB SHARE API

```zig
// src/browser/share.zig
pub const Share = struct {
    /// Check if Web Share API is available
    pub fn isAvailable() bool {
        return js_share_available() != 0;
    }

    /// Share content (requires user gesture)
    pub fn share(data: ShareData) Effect(void) {
        return Effect.browser(.{
            .operation = .share,
            .title = data.title,
            .text = data.text,
            .url = data.url,
        });
    }

    pub const ShareData = struct {
        title: ?[]const u8 = null,
        text: ?[]const u8 = null,
        url: ?[]const u8 = null,
    };
};
```

---

## PERFORMANCE API

```zig
// src/browser/performance.zig
pub const Performance = struct {
    /// High-resolution timestamp (milliseconds)
    pub fn now() f64 {
        return js_performance_now();
    }

    /// Create a performance mark
    pub fn mark(name: []const u8) void {
        js_performance_mark(name.ptr, name.len);
    }

    /// Measure between two marks
    pub fn measure(name: []const u8, start_mark: []const u8, end_mark: []const u8) void {
        js_performance_measure(
            name.ptr, name.len,
            start_mark.ptr, start_mark.len,
            end_mark.ptr, end_mark.len,
        );
    }

    /// Get memory info (if available)
    pub fn memory() ?MemoryInfo {
        if (js_performance_memory_available() == 0) return null;
        return .{
            .used_js_heap_size = js_performance_memory_used(),
            .total_js_heap_size = js_performance_memory_total(),
        };
    }

    pub const MemoryInfo = struct {
        used_js_heap_size: usize,
        total_js_heap_size: usize,
    };
};

/// Intersection Observer for lazy loading
pub const IntersectionObserver = struct {
    handle: u32,

    pub fn init(options: Options, comptime handler: fn ([]const Entry) Msg) IntersectionObserver {
        return .{
            .handle = js_intersection_observer_create(
                options.root_margin.ptr, options.root_margin.len,
                options.threshold,
            ),
        };
    }

    pub fn observe(self: IntersectionObserver, element_id: []const u8) void {
        js_intersection_observer_observe(self.handle, element_id.ptr, element_id.len);
    }

    pub fn unobserve(self: IntersectionObserver, element_id: []const u8) void {
        js_intersection_observer_unobserve(self.handle, element_id.ptr, element_id.len);
    }

    pub fn disconnect(self: IntersectionObserver) void {
        js_intersection_observer_disconnect(self.handle);
    }

    pub const Options = struct {
        root_margin: []const u8 = "0px",
        threshold: f32 = 0.0,
    };

    pub const Entry = struct {
        element_id: []const u8,
        is_intersecting: bool,
        intersection_ratio: f32,
    };
};
```

---

## WEB WORKERS (Parallelism)

```zig
// src/browser/workers.zig
pub const Worker = struct {
    handle: u32,

    /// Spawn a Web Worker running WASM
    pub fn spawn(wasm_url: []const u8) Effect(Worker) {
        return Effect.browser(.{
            .operation = .worker_spawn,
            .url = wasm_url,
        });
    }

    /// Send message to worker
    pub fn postMessage(self: Worker, data: []const u8) void {
        js_worker_post_message(self.handle, data.ptr, data.len);
    }

    /// Terminate worker
    pub fn terminate(self: Worker) void {
        js_worker_terminate(self.handle);
    }

    /// Subscribe to messages from worker
    pub fn onMessage(self: Worker, comptime handler: fn ([]const u8) Msg) Effect.Subscription {
        return .{
            .type = .worker_message,
            .handle = self.handle,
            .handler = handler,
        };
    }
};

// Usage: Offload heavy computation
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .start_heavy_computation => Worker.spawn("/worker.wasm").map(.worker_ready, .worker_error),
        .worker_ready => |worker| {
            model.worker = worker;
            worker.postMessage(model.data_to_process);
            return .none;
        },
        .computation_result => |result| {
            model.result = result;
            model.worker.?.terminate();
            model.worker = null;
            return .none;
        },
    };
}
```

---

## JS RUNTIME IMPLEMENTATION

```javascript
// www/zui-browser.js - Browser API bindings

const ZuiBrowserAPIs = {
    resultBuffer: null,

    // ==================== STORAGE ====================

    js_storage_get(store, keyPtr, keyLen) {
        const key = ZuiRuntime.ptrToStr(keyPtr, keyLen);
        const storage = store === 0 ? localStorage : sessionStorage;
        const value = storage.getItem(key);

        if (value === null) return 0;

        this.resultBuffer = new TextEncoder().encode(value);
        return this.resultBuffer.length;
    },

    js_storage_read(bufPtr, bufLen) {
        if (!this.resultBuffer) return 0;
        new Uint8Array(ZuiRuntime.memory.buffer, bufPtr, bufLen)
            .set(this.resultBuffer);
        const len = this.resultBuffer.length;
        this.resultBuffer = null;
        return len;
    },

    js_storage_set(store, keyPtr, keyLen, valPtr, valLen) {
        const key = ZuiRuntime.ptrToStr(keyPtr, keyLen);
        const value = ZuiRuntime.ptrToStr(valPtr, valLen);
        const storage = store === 0 ? localStorage : sessionStorage;
        storage.setItem(key, value);
    },

    // ... other storage methods

    // ==================== CRYPTO ====================

    js_crypto_get_random_values(bufPtr, len) {
        const buffer = new Uint8Array(ZuiRuntime.memory.buffer, bufPtr, len);
        crypto.getRandomValues(buffer);
    },

    js_crypto_random_uuid(bufPtr) {
        const uuid = crypto.randomUUID();
        const buffer = new Uint8Array(ZuiRuntime.memory.buffer, bufPtr, 36);
        new TextEncoder().encodeInto(uuid, buffer);
    },

    // Async crypto operations go through Effect system
    async handleCryptoEffect(effect) {
        switch (effect.operation) {
            case 'crypto_hash': {
                const data = this.readBuffer(effect.data);
                const hash = await crypto.subtle.digest(effect.algorithm, data);
                return new Uint8Array(hash);
            }
            case 'crypto_encrypt': {
                const key = await this.importKey(effect.key);
                const iv = crypto.getRandomValues(new Uint8Array(12));
                const encrypted = await crypto.subtle.encrypt(
                    { name: 'AES-GCM', iv },
                    key,
                    this.readBuffer(effect.data)
                );
                // Return iv + ciphertext
                const result = new Uint8Array(12 + encrypted.byteLength);
                result.set(iv);
                result.set(new Uint8Array(encrypted), 12);
                return result;
            }
            // ... other crypto operations
        }
    },

    // ==================== WEBSOCKET ====================

    sockets: new Map(),
    nextSocketId: 1,

    handleWebSocketEffect(effect) {
        const id = this.nextSocketId++;
        const ws = new WebSocket(effect.url);

        ws.onopen = () => {
            ZuiRuntime.wasm.exports.effect_callback(effect.callbackId, 1, id, 0);
        };

        ws.onmessage = (e) => {
            const data = new TextEncoder().encode(e.data);
            const ptr = ZuiRuntime.wasm.exports.alloc(data.length);
            new Uint8Array(ZuiRuntime.memory.buffer, ptr, data.length).set(data);
            ZuiRuntime.wasm.exports.websocket_message(id, ptr, data.length);
        };

        ws.onclose = (e) => {
            ZuiRuntime.wasm.exports.websocket_close(id, e.code);
            this.sockets.delete(id);
        };

        ws.onerror = () => {
            ZuiRuntime.wasm.exports.websocket_error(id);
        };

        this.sockets.set(id, ws);
    },

    // ==================== INTL ====================

    js_intl_format_number(value, localePtr, localeLen, optionsPtr, optionsLen) {
        const locale = ZuiRuntime.ptrToStr(localePtr, localeLen);
        const options = JSON.parse(ZuiRuntime.ptrToStr(optionsPtr, optionsLen));

        const formatted = new Intl.NumberFormat(locale, options).format(value);
        this.resultBuffer = new TextEncoder().encode(formatted);
        return this.resultBuffer.length;
    },

    // ... other Intl methods

    // ==================== MEDIA ====================

    js_media_query_matches(queryPtr, queryLen) {
        const query = ZuiRuntime.ptrToStr(queryPtr, queryLen);
        return window.matchMedia(query).matches ? 1 : 0;
    },

    // ... etc
};

// Merge with main runtime
Object.assign(ZuiRuntime.imports.zui, ZuiBrowserAPIs);
```

---

## SUMMARY TABLE

| API | Sync/Async | Effect Integration | Notes |
|-----|------------|-------------------|-------|
| localStorage | Sync | Optional | ~5MB limit |
| sessionStorage | Sync | Optional | Per-tab |
| IndexedDB | Async | Required | Large data |
| crypto.getRandomValues | Sync | No | Fast |
| crypto.subtle | Async | Required | Encryption, hashing |
| Clipboard | Async | Required | Requires gesture |
| Notifications | Async | Required | Requires permission |
| WebSocket | Async | Required | Real-time |
| BroadcastChannel | Async | Subscription | Multi-tab |
| Geolocation | Async | Required | Requires permission |
| Intl | Sync | No | Formatting |
| Media Queries | Sync | Subscription for changes | Responsive |
| Web Share | Async | Required | Requires gesture |
| Performance | Sync | No | Metrics |
| Web Workers | Async | Required | Parallelism |

---

## CONCLUSION

ZUI provides comprehensive access to browser APIs through a type-safe Zig interface. Synchronous APIs are available directly, while asynchronous APIs integrate with the Effect system for predictable state management.

**Links:**
- [← Previous: Developer Experience](19-developer-experience.md)
- [↑ Back to index](../README.md)
