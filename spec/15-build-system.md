# 15 - BUILD SYSTEM

**Document:** Build and Compilation System
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

The ZUI build system orchestrates the compilation of Zig code to WASM, CSS extraction, asset optimization, and preparation of the final bundle. It supports multiple rendering modes (SPA, SSR, SSG) and deployment to the Fermyon Spin platform.

---

## BUILD.ZIG

### Complete Build Configuration

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // Target: wasm32-freestanding
    const target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });

    // Optimization mode
    const optimize = b.standardOptimizeOption(.{});

    // ============== WASM COMPILATION ==============

    const wasm = b.addExecutable(.{
        .name = "app",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    // WASM-specific configuration
    wasm.entry = .disabled;
    wasm.rdynamic = true;
    wasm.import_memory = true;
    wasm.initial_memory = 65536 * 16;  // 1MB
    wasm.max_memory = 65536 * 256;     // 16MB
    wasm.stack_size = 65536 * 4;       // 256KB

    // Link with standard library (minimal)
    wasm.bundle_compiler_rt = true;

    // ============== CSS EXTRACTION ==============

    const css_extract = b.addSystemCommand(&.{"zig"});
    css_extract.addArgs(&.{
        "run",
        "build/extract_css.zig",
        "--",
        "src",
        "www/styles.css",
    });

    // CSS extraction must run before WASM compilation
    wasm.step.dependOn(&css_extract.step);

    // ============== OPTIMIZATION ==============

    if (optimize == .ReleaseFast or optimize == .ReleaseSmall) {
        // Run wasm-opt if available
        const wasm_opt = b.addSystemCommand(&.{"wasm-opt"});
        wasm_opt.addArgs(&.{
            "-O3",
            "--enable-bulk-memory",
            "--enable-simd",
            "-o",
        });
        wasm_opt.addArtifactArg(wasm);
        wasm_opt.addArg("www/app.wasm");

        wasm_opt.step.dependOn(&wasm.step);
        b.getInstallStep().dependOn(&wasm_opt.step);
    } else {
        // Development build - just copy
        const install_wasm = b.addInstallArtifact(wasm, .{
            .dest_dir = .{ .override = .{ .custom = "../www" } },
        });
        b.getInstallStep().dependOn(&install_wasm.step);
    }

    // ============== ASSET COPYING ==============

    const copy_assets = b.addSystemCommand(&.{"cp"});
    copy_assets.addArgs(&.{
        "-r",
        "assets/",
        "www/assets/",
    });
    b.getInstallStep().dependOn(&copy_assets.step);

    // ============== DEV SERVER ==============

    const run_server = b.step("serve", "Run development server");
    const server = b.addSystemCommand(&.{"python3"});
    server.addArgs(&.{
        "-m",
        "http.server",
        "8080",
        "--directory",
        "www",
    });
    run_server.dependOn(&server.step);

    // ============== TESTS ==============

    const tests = b.addTest(.{
        .root_source_file = .{ .path = "src/main.zig" },
        .target = b.host,
    });

    const run_tests = b.addRunArtifact(tests);
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_tests.step);

    // ============== BENCHMARKS ==============

    const bench = b.addExecutable(.{
        .name = "bench",
        .root_source_file = .{ .path = "bench/main.zig" },
        .target = b.host,
        .optimize = .ReleaseFast,
    });

    const run_bench = b.addRunArtifact(bench);
    const bench_step = b.step("bench", "Run benchmarks");
    bench_step.dependOn(&run_bench.step);

    // ============== CLEAN ==============

    const clean = b.step("clean", "Remove build artifacts");
    const rm_zig_cache = b.addSystemCommand(&.{ "rm", "-rf", "zig-cache" });
    const rm_zig_out = b.addSystemCommand(&.{ "rm", "-rf", "zig-out" });
    const rm_www_wasm = b.addSystemCommand(&.{ "rm", "-f", "www/app.wasm" });
    const rm_www_css = b.addSystemCommand(&.{ "rm", "-f", "www/styles.css" });

    clean.dependOn(&rm_zig_cache.step);
    clean.dependOn(&rm_zig_out.step);
    clean.dependOn(&rm_www_wasm.step);
    clean.dependOn(&rm_www_css.step);
}
```

---

## RENDERING MODE CONFIGURATION

### Build Mode Option

```zig
// build.zig - Extended for rendering modes
pub fn build(b: *std.Build) void {
    // Rendering mode selection
    const mode = b.option(
        RenderMode,
        "mode",
        "Rendering mode: spa, ssr, or ssg",
    ) orelse .spa;

    const RenderMode = enum { spa, ssr, ssg };

    switch (mode) {
        .spa => buildSPA(b),
        .ssr => buildSSR(b),
        .ssg => buildSSG(b),
    }
}

fn buildSPA(b: *std.Build) void {
    // Client-only WASM (wasm32-freestanding)
    const client_target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });

    const wasm = b.addExecutable(.{
        .name = "app",
        .root_source_file = .{ .path = "src/client.zig" },
        .target = client_target,
        .optimize = b.standardOptimizeOption(.{}),
    });

    configureClientWasm(wasm);
    b.installArtifact(wasm);
}

fn buildSSR(b: *std.Build) void {
    // Server WASM (wasm32-wasi) for Fermyon Spin
    const server_target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .wasi,
    });

    const server_wasm = b.addExecutable(.{
        .name = "server",
        .root_source_file = .{ .path = "src/server.zig" },
        .target = server_target,
        .optimize = b.standardOptimizeOption(.{}),
    });

    // Client WASM for hydration
    const client_target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });

    const client_wasm = b.addExecutable(.{
        .name = "client",
        .root_source_file = .{ .path = "src/client.zig" },
        .target = client_target,
        .optimize = b.standardOptimizeOption(.{}),
    });

    configureClientWasm(client_wasm);

    b.installArtifact(server_wasm);
    b.installArtifact(client_wasm);
}

fn buildSSG(b: *std.Build) void {
    // SSG generator runs on host
    const generator = b.addExecutable(.{
        .name = "ssg-generator",
        .root_source_file = .{ .path = "src/ssg.zig" },
        .target = b.host,
        .optimize = .ReleaseFast,
    });

    // Run generator to produce static HTML
    const run_generator = b.addRunArtifact(generator);
    run_generator.addArgs(&.{ "www/", "src/pages/" });

    // Optional client WASM for hydration
    const client_target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });

    const client_wasm = b.addExecutable(.{
        .name = "client",
        .root_source_file = .{ .path = "src/client.zig" },
        .target = client_target,
        .optimize = b.standardOptimizeOption(.{}),
    });

    configureClientWasm(client_wasm);

    const ssg_step = b.step("ssg", "Generate static site");
    ssg_step.dependOn(&run_generator.step);
    ssg_step.dependOn(&client_wasm.step);
}

fn configureClientWasm(wasm: *std.Build.Step.Compile) void {
    wasm.entry = .disabled;
    wasm.rdynamic = true;
    wasm.import_memory = true;
    wasm.initial_memory = 65536 * 16;  // 1MB
    wasm.max_memory = 65536 * 256;     // 16MB
    wasm.stack_size = 65536 * 4;       // 256KB
    wasm.bundle_compiler_rt = true;
}
```

---

## FERMYON SPIN INTEGRATION

### spin.toml Generation

```zig
// build/spin_config.zig
const std = @import("std");

pub fn generateSpinToml(
    allocator: std.mem.Allocator,
    mode: RenderMode,
    app_name: []const u8,
) ![]const u8 {
    var buffer = std.ArrayList(u8).init(allocator);
    const writer = buffer.writer();

    try writer.print(
        \\spin_manifest_version = 2
        \\
        \\[application]
        \\name = "{s}"
        \\version = "1.0.0"
        \\
    , .{app_name});

    switch (mode) {
        .spa => try writeSPAConfig(writer),
        .ssr => try writeSSRConfig(writer),
        .ssg => try writeSSGConfig(writer),
    }

    return buffer.toOwnedSlice();
}

fn writeSPAConfig(writer: anytype) !void {
    try writer.writeAll(
        \\# SPA Mode - Static file serving only
        \\[[trigger.http]]
        \\route = "/..."
        \\component = "static"
        \\
        \\[component.static]
        \\source = { url = "https://github.com/fermyon/spin-fileserver/releases/download/v0.3.0/spin_static_fs.wasm", digest = "sha256:..." }
        \\files = [{ source = "www/", destination = "/" }]
        \\
    );
}

fn writeSSRConfig(writer: anytype) !void {
    try writer.writeAll(
        \\# SSR Mode - Dynamic server + static assets
        \\[[trigger.http]]
        \\route = "/api/..."
        \\component = "api"
        \\
        \\[[trigger.http]]
        \\route = "/static/..."
        \\component = "static"
        \\
        \\[[trigger.http]]
        \\route = "/..."
        \\component = "ssr"
        \\
        \\[component.api]
        \\source = "target/wasm32-wasi/release/server.wasm"
        \\allowed_outbound_hosts = ["https://*"]
        \\
        \\[component.ssr]
        \\source = "target/wasm32-wasi/release/server.wasm"
        \\
        \\[component.static]
        \\source = { url = "https://github.com/fermyon/spin-fileserver/releases/download/v0.3.0/spin_static_fs.wasm", digest = "sha256:..." }
        \\files = [{ source = "www/", destination = "/" }]
        \\
    );
}

fn writeSSGConfig(writer: anytype) !void {
    try writer.writeAll(
        \\# SSG Mode - Pre-rendered static files
        \\[[trigger.http]]
        \\route = "/..."
        \\component = "static"
        \\
        \\[component.static]
        \\source = { url = "https://github.com/fermyon/spin-fileserver/releases/download/v0.3.0/spin_static_fs.wasm", digest = "sha256:..." }
        \\files = [{ source = "www/", destination = "/" }]
        \\
    );
}
```

### Build Step for Spin

```zig
// In build.zig
const spin_build = b.step("spin", "Build for Fermyon Spin deployment");

// Generate spin.toml
const gen_spin_toml = b.addSystemCommand(&.{"zig"});
gen_spin_toml.addArgs(&.{
    "run",
    "build/gen_spin_toml.zig",
    "--",
});
gen_spin_toml.addArg(@tagName(mode));

spin_build.dependOn(&gen_spin_toml.step);
spin_build.dependOn(b.getInstallStep());
```

---

## BUILD STEPS

### Development Workflow

```bash
# Clean build
zig build clean

# Development build (unoptimized, SPA mode default)
zig build

# Build with specific rendering mode
zig build -Dmode=spa    # Client-only SPA
zig build -Dmode=ssr    # Server + Client for SSR
zig build -Dmode=ssg    # Static site generation

# Run dev server
zig build serve

# Run tests
zig build test

# Run benchmarks
zig build bench
```

### Production Build

```bash
# Optimized SPA build
zig build -Doptimize=ReleaseSmall -Dmode=spa

# Optimized SSR build
zig build -Doptimize=ReleaseSmall -Dmode=ssr

# Optimized SSG build
zig build -Doptimize=ReleaseSmall -Dmode=ssg

# This produces:
# - www/app.wasm (optimized with wasm-opt) - for SPA/SSG client
# - www/client.wasm - for SSR hydration
# - target/wasm32-wasi/release/server.wasm - for SSR server
# - www/styles.css (extracted and minified)
# - www/assets/* (copied)
# - www/*.html (for SSG mode)
```

### Fermyon Spin Deployment

```bash
# Build for Spin (generates spin.toml)
zig build spin -Dmode=spa    # Static SPA deployment
zig build spin -Dmode=ssr    # Dynamic SSR deployment
zig build spin -Dmode=ssg    # Static SSG deployment

# Local development with Spin
spin up

# Deploy to Fermyon Cloud
spin deploy
```

---

## CSS EXTRACTION INTEGRATION

### Extract Script

```zig
// build/extract_css.zig
const std = @import("std");
const CssExtractor = @import("css_extractor.zig").CssExtractor;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    var args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    if (args.len < 3) {
        std.debug.print("Usage: extract_css <source_dir> <output_file>\n", .{});
        return error.InvalidArgs;
    }

    const source_dir = args[1];
    const output_file = args[2];

    std.debug.print("Extracting CSS from {s}...\n", .{source_dir});

    var extractor = CssExtractor.init(allocator);
    defer extractor.deinit();

    // Walk source directory
    var dir = try std.fs.cwd().openIterableDir(source_dir, .{});
    defer dir.close();

    var walker = try dir.walk(allocator);
    defer walker.deinit();

    var file_count: u32 = 0;
    while (try walker.next()) |entry| {
        if (entry.kind == .file and std.mem.endsWith(u8, entry.path, ".zig")) {
            const full_path = try std.fs.path.join(
                allocator,
                &.{ source_dir, entry.path },
            );
            defer allocator.free(full_path);

            try extractor.extractFromFile(full_path);
            file_count += 1;
        }
    }

    // Write CSS
    try extractor.writeCss(output_file);

    // Write source map
    const source_map_path = try std.fmt.allocPrint(
        allocator,
        "{s}.map",
        .{output_file},
    );
    defer allocator.free(source_map_path);
    try extractor.writeSourceMap(source_map_path);

    const stats = extractor.getStats();

    std.debug.print("✓ Processed {d} files\n", .{file_count});
    std.debug.print("✓ Generated {d} CSS classes\n", .{stats.total_classes});
    std.debug.print("✓ Average reuse: {d:.2}x\n", .{stats.avg_reuse});
    std.debug.print("✓ Output: {s}\n", .{output_file});
}
```

---

## OPTIMIZATION STRATEGIES

### WASM Optimization

```bash
# Using wasm-opt (from binaryen)
wasm-opt -O3 \
  --enable-bulk-memory \
  --enable-simd \
  --enable-nontrapping-float-to-int \
  -o app.optimized.wasm \
  app.wasm

# Size comparison
ls -lh app.wasm app.optimized.wasm
```

### CSS Minification

```bash
# Using csso-cli
csso -i styles.css -o styles.min.css

# Or integrate in build.zig
const minify_css = b.addSystemCommand(&.{"csso"});
minify_css.addArgs(&.{ "-i", "www/styles.css", "-o", "www/styles.min.css" });
```

### Gzip Compression

```bash
# Pre-compress for static serving
gzip -9 -k www/app.wasm      # -> app.wasm.gz
gzip -9 -k www/styles.css    # -> styles.css.gz
```

---

## DEVELOPMENT SERVER

### Python Simple Server

```python
#!/usr/bin/env python3
# build/serve.py
import http.server
import socketserver
import os

PORT = 8080
DIRECTORY = "www"

class MyHTTPRequestHandler(http.server.SimpleHTTPRequestHandler):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, directory=DIRECTORY, **kwargs)

    def end_headers(self):
        # CORS headers for development
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Cross-Origin-Opener-Policy', 'same-origin')
        self.send_header('Cross-Origin-Embedder-Policy', 'require-corp')

        # WASM MIME type
        if self.path.endswith('.wasm'):
            self.send_header('Content-Type', 'application/wasm')

        super().end_headers()

with socketserver.TCPServer(("", PORT), MyHTTPRequestHandler) as httpd:
    print(f"Server running at http://localhost:{PORT}/")
    print("Press Ctrl+C to stop")
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        print("\nShutting down server...")
```

### With Live Reload

```python
# build/serve_watch.py
import time
import subprocess
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class RebuildHandler(FileSystemEventHandler):
    def on_modified(self, event):
        if event.src_path.endswith('.zig'):
            print(f"File changed: {event.src_path}")
            print("Rebuilding...")
            subprocess.run(["zig", "build"])
            print("✓ Build complete")

observer = Observer()
observer.schedule(RebuildHandler(), path="src", recursive=True)
observer.start()

print("Watching for changes...")
try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    observer.stop()
observer.join()
```

---

## CONTINUOUS INTEGRATION

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Setup Zig
      uses: goto-bus-stop/setup-zig@v2
      with:
        version: 0.15.x

    - name: Build
      run: zig build -Doptimize=ReleaseSmall

    - name: Test
      run: zig build test

    - name: Check size
      run: |
        wasm_size=$(stat -f%z www/app.wasm)
        echo "WASM size: $(($wasm_size / 1024)) KB"
        if [ $wasm_size -gt 81920 ]; then
          echo "Warning: WASM bundle exceeds 80KB target"
        fi

    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: build
        path: |
          www/app.wasm
          www/styles.css
```

---

## BUNDLING FOR PRODUCTION

### Production Checklist

```bash
#!/bin/bash
# build/prod.sh

set -e

echo "🏗️  Production Build"
echo "===================="

# Clean
echo "Cleaning..."
zig build clean

# Build optimized
echo "Building WASM..."
zig build -Doptimize=ReleaseSmall

# Optimize with wasm-opt
echo "Optimizing WASM..."
wasm-opt -O3 \
  --enable-bulk-memory \
  --enable-simd \
  www/app.wasm \
  -o www/app.wasm

# Minify CSS
echo "Minifying CSS..."
csso -i www/styles.css -o www/styles.min.css
mv www/styles.min.css www/styles.css

# Pre-compress
echo "Compressing..."
gzip -9 -k www/app.wasm
gzip -9 -k www/styles.css
gzip -9 -k www/index.html

# Report sizes
echo ""
echo "📦 Bundle Sizes:"
echo "---------------"
ls -lh www/app.wasm* | awk '{print "WASM: " $5 " (" $9 ")"}'
ls -lh www/styles.css* | awk '{print "CSS:  " $5 " (" $9 ")"}'

echo ""
echo "✅ Production build complete!"
```

---

## DEBUGGING BUILD ISSUES

### Common Issues

```bash
# Issue: "undefined symbol" errors
# Solution: Ensure all functions are exported
export fn myFunction() void { ... }

# Issue: Memory allocation failures
# Solution: Increase initial_memory
wasm.initial_memory = 65536 * 32; // 2MB

# Issue: Stack overflow
# Solution: Increase stack_size
wasm.stack_size = 65536 * 8; // 512KB

# Issue: CSS extraction fails
# Solution: Check syntax in .zig files
zig fmt --check src/

# Issue: WASM too large
# Solution: Use ReleaseSmall optimization
zig build -Doptimize=ReleaseSmall
```

---

## CONCLUSION

The ZUI build system automates compilation, CSS extraction, optimization, and packaging, providing efficient workflows for both development and production. It supports multiple rendering modes (SPA, SSR, SSG) and seamless deployment to the Fermyon Spin platform.

**Links:**
- [← Previous: JS Runtime](14-js-runtime.md)
- [→ Next: Examples](16-examples.md)
- [→ Fermyon Spin Deployment](18-fermyon-spin.md)
- [↑ Back to index](../README.md)
