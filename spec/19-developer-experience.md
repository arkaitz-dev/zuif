# 19 - DEVELOPER EXPERIENCE

**Document:** Developer Experience and Tooling
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

Developer experience (DX) is critical for ZUI adoption. This document specifies the development workflow, hot reload capabilities, debugging tools, and error handling that make ZUI productive to use.

---

## DEVELOPMENT WORKFLOW

### Basic Cycle

```
┌─────────────────────────────────────────────────────────────┐
│                    DEVELOPMENT LOOP                         │
│                                                             │
│   Edit .zig ──▶ Save ──▶ Auto-compile ──▶ Browser updates  │
│       ▲                                          │          │
│       └──────────────────────────────────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Commands

```bash
# Start development mode
zig build dev

# This starts:
# 1. File watcher on src/
# 2. Auto-recompilation on changes
# 3. Development server on :8080
# 4. WebSocket server for live reload
# 5. Opens browser automatically (optional)
```

---

## HOT RELOAD SYSTEM

### Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Zig Files  │────▶│   Watcher    │────▶│  Compiler    │
│   (src/*.zig)│     │  (inotify)   │     │  (zig build) │
└──────────────┘     └──────────────┘     └──────────────┘
                                                 │
                                                 ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Browser    │◀────│  WebSocket   │◀────│  Dev Server  │
│   (reload)   │     │  (notify)    │     │  (http+ws)   │
└──────────────┘     └──────────────┘     └──────────────┘
```

### Phase 1: Smart Reload (v0.1)

Preserves application state across reloads.

```javascript
// www/zui-dev.js - Development runtime

const ZuiDev = {
    ws: null,

    init() {
        // Connect to dev server WebSocket
        this.ws = new WebSocket(`ws://${location.host}/__zui_dev`);

        this.ws.onmessage = (event) => {
            const msg = JSON.parse(event.data);

            switch (msg.type) {
                case 'reload':
                    this.handleReload(msg);
                    break;
                case 'error':
                    this.showCompileError(msg.error);
                    break;
            }
        };

        // Check for preserved state on load
        this.restoreState();
    },

    handleReload(msg) {
        // Preserve current state
        this.preserveState();

        // Reload page
        location.reload();
    },

    preserveState() {
        try {
            // Get state from WASM
            const statePtr = ZuiRuntime.wasm.exports.getState();
            const stateLen = ZuiRuntime.wasm.exports.getStateLen();
            const state = ZuiRuntime.ptrToStr(statePtr, stateLen);

            // Also preserve scroll position and focus
            const uiState = {
                model: state,
                scrollX: window.scrollX,
                scrollY: window.scrollY,
                focusedId: document.activeElement?.id || null,
                route: location.pathname,
            };

            sessionStorage.setItem('__ZUI_DEV_STATE__', JSON.stringify(uiState));
        } catch (e) {
            console.warn('Could not preserve state:', e);
        }
    },

    restoreState() {
        const saved = sessionStorage.getItem('__ZUI_DEV_STATE__');
        if (!saved) return;

        sessionStorage.removeItem('__ZUI_DEV_STATE__');

        try {
            const uiState = JSON.parse(saved);

            // Initialize WASM with preserved model
            ZuiRuntime.initWithState(uiState.model);

            // Restore scroll position
            requestAnimationFrame(() => {
                window.scrollTo(uiState.scrollX, uiState.scrollY);

                // Restore focus
                if (uiState.focusedId) {
                    document.getElementById(uiState.focusedId)?.focus();
                }
            });

            console.log('%c[ZUI Dev] State restored', 'color: green');

        } catch (e) {
            console.warn('Could not restore state, starting fresh:', e);
            ZuiRuntime.init();
        }
    },

    showCompileError(error) {
        // Show error overlay
        const overlay = document.createElement('div');
        overlay.id = '__zui_error_overlay';
        overlay.innerHTML = `
            <style>
                #__zui_error_overlay {
                    position: fixed;
                    inset: 0;
                    background: rgba(0, 0, 0, 0.9);
                    color: #ff6b6b;
                    font-family: monospace;
                    padding: 2rem;
                    overflow: auto;
                    z-index: 99999;
                }
                #__zui_error_overlay h1 {
                    color: #ff6b6b;
                    margin: 0 0 1rem 0;
                }
                #__zui_error_overlay pre {
                    background: #1a1a1a;
                    padding: 1rem;
                    border-radius: 4px;
                    overflow-x: auto;
                    white-space: pre-wrap;
                }
                #__zui_error_overlay .dismiss {
                    position: absolute;
                    top: 1rem;
                    right: 1rem;
                    background: #333;
                    color: white;
                    border: none;
                    padding: 0.5rem 1rem;
                    cursor: pointer;
                }
            </style>
            <button class="dismiss" onclick="this.parentElement.remove()">Dismiss</button>
            <h1>Compilation Error</h1>
            <pre>${escapeHtml(error)}</pre>
        `;

        // Remove existing overlay if present
        document.getElementById('__zui_error_overlay')?.remove();
        document.body.appendChild(overlay);
    }
};

// Auto-init in development
if (location.hostname === 'localhost') {
    ZuiDev.init();
}
```

### WASM State Export

```zig
// src/runtime/dev.zig
const std = @import("std");
const app = @import("app.zig");

var state_buffer: [65536]u8 = undefined;
var state_len: usize = 0;

/// Export current model state as JSON
export fn getState() [*]const u8 {
    var stream = std.io.fixedBufferStream(&state_buffer);
    std.json.stringify(app.model, .{}, stream.writer()) catch {
        return "{}";
    };
    state_len = stream.pos;
    return &state_buffer;
}

export fn getStateLen() usize {
    return state_len;
}
```

### Phase 2: WASM Hot Swap (v0.2+)

True hot reload without page refresh.

```javascript
// www/zui-dev.js - Hot swap implementation

const ZuiHotSwap = {
    async hotSwap(newWasmUrl) {
        console.log('%c[ZUI Dev] Hot swapping WASM...', 'color: orange');

        try {
            // 1. Capture current state
            const statePtr = ZuiRuntime.wasm.exports.getState();
            const stateLen = ZuiRuntime.wasm.exports.getStateLen();
            const stateJson = ZuiRuntime.ptrToStr(statePtr, stateLen);

            // 2. Fetch new WASM (with cache bust)
            const response = await fetch(newWasmUrl + '?t=' + Date.now());
            if (!response.ok) throw new Error('Failed to fetch new WASM');

            const bytes = await response.arrayBuffer();

            // 3. Instantiate new module with same imports
            const imports = ZuiRuntime.createImports();
            const { instance } = await WebAssembly.instantiate(bytes, imports);

            // 4. Transfer state to new WASM
            const stateBytes = new TextEncoder().encode(stateJson);
            const ptr = instance.exports.alloc(stateBytes.length);
            new Uint8Array(instance.exports.memory.buffer, ptr, stateBytes.length)
                .set(stateBytes);

            // 5. Initialize new WASM with transferred state
            instance.exports.initWithState(ptr, stateBytes.length);

            // 6. Replace runtime reference
            const oldWasm = ZuiRuntime.wasm;
            ZuiRuntime.wasm = instance;
            ZuiRuntime.memory = instance.exports.memory;

            // 7. Re-render (view function may have changed)
            instance.exports.render();

            // 8. Old WASM will be garbage collected
            console.log('%c[ZUI Dev] Hot swap complete!', 'color: green');

        } catch (e) {
            console.error('[ZUI Dev] Hot swap failed, falling back to reload:', e);
            ZuiDev.handleReload({ type: 'reload' });
        }
    }
};
```

### Hot Swap Limitations

| Scenario | Behavior |
|----------|----------|
| Model structure unchanged | Hot swap works perfectly |
| Model fields added | Deserialization uses defaults for new fields |
| Model fields removed | Extra fields ignored |
| Model field types changed | **Falls back to page reload** |
| View function changed | Hot swap + re-render |
| New routes added | Hot swap works |
| WASM exports changed | **Falls back to page reload** |

---

## DEVELOPMENT SERVER

### Implementation

```zig
// build/dev_server.zig
const std = @import("std");
const http = std.http;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    // Start file watcher in separate thread
    const watcher_thread = try std.Thread.spawn(.{}, watchFiles, .{allocator});
    defer watcher_thread.join();

    // Start HTTP + WebSocket server
    var server = http.Server.init(allocator, .{});
    defer server.deinit();

    const address = std.net.Address.parseIp("127.0.0.1", 8080) catch unreachable;
    try server.listen(address);

    std.debug.print("Development server running at http://localhost:8080\n", .{});
    std.debug.print("WebSocket endpoint: ws://localhost:8080/__zui_dev\n", .{});

    // Accept connections
    while (true) {
        var connection = try server.accept();
        _ = try std.Thread.spawn(.{}, handleConnection, .{connection});
    }
}

fn watchFiles(allocator: std.mem.Allocator) void {
    // Platform-specific file watching
    // Linux: inotify
    // macOS: FSEvents
    // Windows: ReadDirectoryChangesW

    // On change:
    // 1. Run `zig build`
    // 2. If success: broadcast "reload" to all WebSocket clients
    // 3. If error: broadcast error message
}
```

### Alternative: Python Dev Server (Simpler)

```python
#!/usr/bin/env python3
# build/dev_server.py

import asyncio
import subprocess
import json
from pathlib import Path
from aiohttp import web
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# WebSocket clients
clients = set()

class ZigWatcher(FileSystemEventHandler):
    def __init__(self, loop):
        self.loop = loop
        self.debounce_task = None

    def on_modified(self, event):
        if not event.src_path.endswith('.zig'):
            return

        # Debounce rapid changes
        if self.debounce_task:
            self.debounce_task.cancel()

        self.debounce_task = self.loop.call_later(
            0.1,
            lambda: asyncio.create_task(self.rebuild())
        )

    async def rebuild(self):
        print(f"Rebuilding...")

        result = subprocess.run(
            ['zig', 'build'],
            capture_output=True,
            text=True
        )

        if result.returncode == 0:
            print("Build successful, notifying clients...")
            await broadcast({'type': 'reload'})
        else:
            print(f"Build failed:\n{result.stderr}")
            await broadcast({
                'type': 'error',
                'error': result.stderr
            })

async def broadcast(message):
    if clients:
        msg = json.dumps(message)
        await asyncio.gather(
            *[client.send_str(msg) for client in clients],
            return_exceptions=True
        )

async def websocket_handler(request):
    ws = web.WebSocketResponse()
    await ws.prepare(request)

    clients.add(ws)
    print(f"Client connected ({len(clients)} total)")

    try:
        async for msg in ws:
            pass  # We only send, don't receive
    finally:
        clients.discard(ws)
        print(f"Client disconnected ({len(clients)} total)")

    return ws

async def static_handler(request):
    path = request.path
    if path == '/':
        path = '/index.html'

    file_path = Path('www') / path.lstrip('/')

    if file_path.exists():
        return web.FileResponse(file_path)
    else:
        # SPA fallback
        return web.FileResponse('www/index.html')

def main():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    # Setup file watcher
    watcher = ZigWatcher(loop)
    observer = Observer()
    observer.schedule(watcher, 'src', recursive=True)
    observer.start()

    # Setup web server
    app = web.Application()
    app.router.add_get('/__zui_dev', websocket_handler)
    app.router.add_get('/{path:.*}', static_handler)

    print("Starting development server...")
    print("  http://localhost:8080")
    print("  Watching src/ for changes")
    print()

    web.run_app(app, host='localhost', port=8080, print=None)

if __name__ == '__main__':
    main()
```

---

## ERROR HANDLING

### Compile-Time Errors

Displayed as overlay in browser:

```
┌─────────────────────────────────────────────────────────────┐
│  ╔═══════════════════════════════════════════════════════╗  │
│  ║  COMPILATION ERROR                              [X]   ║  │
│  ╠═══════════════════════════════════════════════════════╣  │
│  ║                                                       ║  │
│  ║  src/app.zig:42:15: error: expected ';'              ║  │
│  ║                                                       ║  │
│  ║      return h.div(.{                                 ║  │
│  ║                 ^                                     ║  │
│  ║                                                       ║  │
│  ║  src/app.zig:41:5: note: to match this '('          ║  │
│  ║                                                       ║  │
│  ╚═══════════════════════════════════════════════════════╝  │
│                                                             │
│  [Waiting for fix...]                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Runtime Errors

Captured and displayed with stack trace:

```javascript
// Error boundary in JS runtime
ZuiRuntime.setErrorHandler((error) => {
    // In development: show overlay
    if (isDev) {
        ZuiDev.showRuntimeError({
            message: error.message,
            stack: error.stack,
            wasmStack: ZuiRuntime.getWasmStackTrace(),
        });
    }

    // In production: report to error service
    else {
        reportError(error);
    }
});
```

---

## SOURCE MAPS

### WASM Debug Info

```zig
// build.zig - Development build includes debug info
const wasm = b.addExecutable(.{
    .name = "app",
    .root_source_file = .{ .path = "src/main.zig" },
    .target = wasm_target,
    .optimize = if (is_dev) .Debug else .ReleaseSmall,
});

// Generate DWARF debug info for development
if (is_dev) {
    wasm.dwarf_debug_info = true;
}
```

### CSS Source Maps

```json
// www/styles.css.map
{
    "version": 3,
    "sources": ["../src/components/button.zig", "../src/components/card.zig"],
    "mappings": "AAAA,..."
}
```

---

## BROWSER DEVTOOLS INTEGRATION

### Custom Panel (Future)

```javascript
// ZUI DevTools Extension
chrome.devtools.panels.create(
    "ZUI",
    "icon.png",
    "panel.html",
    (panel) => {
        // Panel shows:
        // - Current Model state (tree view)
        // - Message history
        // - VDOM visualization
        // - Performance metrics
    }
);
```

### Console Helpers

```javascript
// Available in development mode
window.__ZUI__ = {
    // Inspect current model
    getModel: () => JSON.parse(ZuiRuntime.getState()),

    // Dispatch message manually
    dispatch: (msgId) => ZuiRuntime.wasm.exports.dispatch(msgId),

    // Get performance metrics
    getMetrics: () => ZuiRuntime.getMetrics(),

    // Force re-render
    render: () => ZuiRuntime.wasm.exports.render(),

    // Get current VDOM (serialized)
    getVdom: () => JSON.parse(ZuiRuntime.getVdom()),
};

console.log('%c[ZUI Dev] Helpers available at window.__ZUI__', 'color: blue');
```

---

## PERFORMANCE PROFILING

### Frame Timing

```javascript
// Automatic performance warnings
ZuiRuntime.onRender((duration) => {
    if (duration > 16) {
        console.warn(
            `%c[ZUI] Slow render: ${duration.toFixed(2)}ms (target: 16ms)`,
            'color: orange'
        );
    }
});
```

### Memory Tracking

```zig
// Debug build tracks allocations
pub fn trackAllocation(size: usize, source: []const u8) void {
    if (comptime std.debug.runtime_safety) {
        allocation_log.append(.{
            .size = size,
            .source = source,
            .timestamp = getTimestamp(),
        });
    }
}
```

---

## DEVELOPMENT VS PRODUCTION

| Feature | Development | Production |
|---------|-------------|------------|
| Hot reload | Enabled | Disabled |
| Error overlays | Full details | Minimal |
| Source maps | Included | Optional |
| Console helpers | Available | Stripped |
| WASM optimization | Debug | ReleaseSmall |
| Bundle size | ~200KB | <80KB |
| Build time | ~1s | ~5s |

### Build Mode Detection

```zig
// In Zig code
const is_dev = @import("builtin").mode == .Debug;

if (is_dev) {
    // Development-only code
    exportDebugHelpers();
}
```

```javascript
// In JS runtime
const isDev = location.hostname === 'localhost'
           || location.hostname === '127.0.0.1';
```

---

## DEVELOPMENT PHASES

### Phase 0 (v0.1): Basic DX
- [x] File watcher with auto-recompilation
- [x] Development server with static files
- [x] Compile error overlay
- [x] Smart Reload (preserves state)

### Phase 1 (v0.2): Enhanced DX
- [ ] WASM Hot Swap (no page reload)
- [ ] Runtime error overlay with WASM stack
- [ ] Console helpers (`window.__ZUI__`)
- [ ] Performance warnings

### Phase 2 (v1.0): Full DX
- [ ] CSS source maps
- [ ] Browser DevTools extension
- [ ] Memory profiling
- [ ] Time-travel debugging

---

## CONCLUSION

A productive developer experience is essential for ZUI adoption. The hot reload system, with Smart Reload in v0.1 and WASM Hot Swap in v0.2, ensures rapid iteration without losing application state.

**Links:**
- [← Previous: Fermyon Spin](18-fermyon-spin.md)
- [↑ Back to index](../README.md)
