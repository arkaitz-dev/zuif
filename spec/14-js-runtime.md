# 14 - JS RUNTIME

**Document:** Enhanced JavaScript Runtime
**Version:** 6.0.0
**Status:** Final

---

## OVERVIEW

The ZUI v6 JavaScript runtime provides update batching, error handling, event management, and memory leak prevention.

---

## ARCHITECTURE

```javascript
// www/zui.js
const ZuiRuntime = {
    // WASM instance
    wasm: null,
    memory: null,

    // Update batching
    updateScheduled: false,
    pendingUpdates: [],

    // Error handling
    errorBoundary: null,

    // Event delegation
    eventHandlers: new Map(),

    // Performance tracking
    metrics: {
        renderTime: [],
        updateTime: [],
    },

    async init(wasmPath) {
        try {
            // Load WASM
            const response = await fetch(wasmPath);
            const bytes = await response.arrayBuffer();

            const imports = this.createImports();
            const result = await WebAssembly.instantiate(bytes, imports);

            this.wasm = result.instance;
            this.memory = this.wasm.exports.memory;

            // Initialize event delegation
            this.setupEventDelegation();

            // Initialize app
            this.wasm.exports.init();

            // Initial render
            this.scheduleUpdate();

        } catch (err) {
            this.handleError(err);
        }
    },

    createImports() {
        return {
            zui: {
                // DOM operations
                js_createElement: this.createElement.bind(this),
                js_createTextNode: this.createTextNode.bind(this),
                js_appendChild: this.appendChild.bind(this),
                js_removeChild: this.removeChild.bind(this),
                js_setAttribute: this.setAttribute.bind(this),
                js_setTextContent: this.setTextContent.bind(this),

                // Effects
                js_setTimeout: this.setTimeout.bind(this),
                js_setInterval: this.setInterval.bind(this),
                js_httpRequest: this.httpRequest.bind(this),

                // Router
                js_pushState: this.pushState.bind(this),
                js_goBack: () => window.history.back(),
                js_goForward: () => window.history.forward(),

                // Console
                js_log: this.log.bind(this),
                js_error: this.error.bind(this),
            },
        };
    },

    // ============== UPDATE BATCHING ==============

    scheduleUpdate() {
        if (this.updateScheduled) return;

        this.updateScheduled = true;
        requestAnimationFrame(() => {
            this.performUpdate();
        });
    },

    performUpdate() {
        const start = performance.now();

        try {
            // Render
            this.wasm.exports.render();

            // Track performance
            const duration = performance.now() - start;
            this.metrics.renderTime.push(duration);

            // Keep only last 100 samples
            if (this.metrics.renderTime.length > 100) {
                this.metrics.renderTime.shift();
            }

            // Warn if slow
            if (duration > 16) {
                console.warn(`Slow render: ${duration.toFixed(2)}ms`);
            }

        } catch (err) {
            this.handleError(err);
        } finally {
            this.updateScheduled = false;
        }
    },

    // ============== DOM OPERATIONS ==============

    createElement(tagPtr, tagLen) {
        const tag = this.ptrToStr(tagPtr, tagLen);
        const element = document.createElement(tag);
        const id = this.registerElement(element);
        return id;
    },

    createTextNode(textPtr, textLen) {
        const text = this.ptrToStr(textPtr, textLen);
        const node = document.createTextNode(text);
        const id = this.registerElement(node);
        return id;
    },

    appendChild(parentId, childId) {
        const parent = this.getElement(parentId);
        const child = this.getElement(childId);

        if (!parent || !child) {
            console.error('Invalid parent or child ID');
            return;
        }

        parent.appendChild(child);
    },

    removeChild(parentId, childId) {
        const parent = this.getElement(parentId);
        const child = this.getElement(childId);

        if (!parent || !child) {
            console.error('Invalid parent or child ID');
            return;
        }

        parent.removeChild(child);
        this.unregisterElement(childId);
    },

    setAttribute(nodeId, keyPtr, keyLen, valPtr, valLen) {
        const element = this.getElement(nodeId);
        if (!element) return;

        const key = this.ptrToStr(keyPtr, keyLen);
        const value = this.ptrToStr(valPtr, valLen);

        // Handle events specially
        if (key.startsWith('on')) {
            const eventType = key.substring(2).toLowerCase();
            this.attachEvent(nodeId, eventType, value);
        } else {
            element.setAttribute(key, value);
        }
    },

    setTextContent(nodeId, textPtr, textLen) {
        const node = this.getElement(nodeId);
        if (!node) return;

        const text = this.ptrToStr(textPtr, textLen);
        node.textContent = text;
    },

    // ============== EVENT HANDLING ==============

    setupEventDelegation() {
        const events = ['click', 'input', 'change', 'submit', 'keydown', 'keyup'];

        events.forEach(eventType => {
            document.addEventListener(eventType, (event) => {
                this.handleDelegatedEvent(eventType, event);
            }, true);
        });
    },

    handleDelegatedEvent(eventType, event) {
        let target = event.target;

        while (target && target !== document) {
            const nodeId = target.dataset?.zuiId;
            if (nodeId) {
                const handler = this.eventHandlers.get(`${nodeId}-${eventType}`);
                if (handler) {
                    event.preventDefault();

                    // Call WASM event handler
                    const msgId = parseInt(handler);
                    this.wasm.exports.dispatch(msgId);

                    // Schedule update
                    this.scheduleUpdate();
                    return;
                }
            }

            target = target.parentElement;
        }
    },

    attachEvent(nodeId, eventType, msgId) {
        const element = this.getElement(nodeId);
        if (!element) return;

        // Set data attribute for event delegation
        element.dataset.zuiId = nodeId;

        // Store handler
        this.eventHandlers.set(`${nodeId}-${eventType}`, msgId);
    },

    // ============== ERROR HANDLING ==============

    handleError(error) {
        console.error('ZUI Runtime Error:', error);

        if (this.errorBoundary) {
            this.errorBoundary(error);
        } else {
            // Default error UI
            document.body.innerHTML = `
                <div style="padding: 20px; background: #fee; border: 2px solid #c00;">
                    <h2>Application Error</h2>
                    <pre>${error.message}\n${error.stack}</pre>
                    <button onclick="location.reload()">Reload</button>
                </div>
            `;
        }
    },

    setErrorBoundary(handler) {
        this.errorBoundary = handler;
    },

    // ============== EFFECTS ==============

    setTimeout(timerId, durationMs) {
        setTimeout(() => {
            try {
                this.wasm.exports.onTimerComplete(timerId);
                this.scheduleUpdate();
            } catch (err) {
                this.handleError(err);
            }
        }, durationMs);
    },

    setInterval(timerId, durationMs) {
        setInterval(() => {
            try {
                this.wasm.exports.onTimerComplete(timerId);
                this.scheduleUpdate();
            } catch (err) {
                this.handleError(err);
            }
        }, durationMs);
    },

    async httpRequest(methodPtr, methodLen, urlPtr, urlLen, bodyPtr, bodyLen) {
        const method = this.ptrToStr(methodPtr, methodLen);
        const url = this.ptrToStr(urlPtr, urlLen);
        const body = bodyPtr ? this.ptrToStr(bodyPtr, bodyLen) : null;

        try {
            const options = {
                method,
                headers: { 'Content-Type': 'application/json' }
            };

            if (body) options.body = body;

            const response = await fetch(url, options);
            const data = await response.text();

            // Call success callback
            const dataPtr = this.strToPtr(data);
            this.wasm.exports.effect_httpSuccess(dataPtr, data.length);

            this.scheduleUpdate();

        } catch (err) {
            // Call error callback
            const msg = err.message;
            const msgPtr = this.strToPtr(msg);
            this.wasm.exports.effect_httpError(msgPtr, msg.length);

            this.scheduleUpdate();
        }
    },

    // ============== UTILITIES ==============

    ptrToStr(ptr, len) {
        const bytes = new Uint8Array(this.memory.buffer, ptr, len);
        return new TextDecoder().decode(bytes);
    },

    strToPtr(str) {
        const bytes = new TextEncoder().encode(str);
        const ptr = this.wasm.exports.alloc(bytes.length);
        new Uint8Array(this.memory.buffer, ptr, bytes.length).set(bytes);
        return ptr;
    },

    // Element registry for memory safety
    nextElementId: 1,
    elements: new Map(),

    registerElement(element) {
        const id = this.nextElementId++;
        this.elements.set(id, element);
        return id;
    },

    getElement(id) {
        return this.elements.get(id);
    },

    unregisterElement(id) {
        this.elements.delete(id);
    },

    // ============== METRICS ==============

    getMetrics() {
        const avgRender = this.metrics.renderTime.reduce((a, b) => a + b, 0) /
                          this.metrics.renderTime.length;

        return {
            avgRenderTime: avgRender.toFixed(2),
            fps: (1000 / avgRender).toFixed(0),
            totalRenders: this.metrics.renderTime.length,
        };
    },
};

// Initialize on load
window.addEventListener('DOMContentLoaded', () => {
    ZuiRuntime.init('app.wasm');
});
```

---

## OPTIMIZATIONS

### 1. Update Batching
- Uses requestAnimationFrame
- Groups multiple updates in one frame
- Target: 60 FPS

### 2. Event Delegation
- One listener per event type
- Avoids memory leaks
- Better performance with many elements

### 3. Memory Safety
- DOM element registry
- Automatic cleanup
- Prevents dangling pointers

### 4. Error Recovery
- Error boundaries
- Fallback UI
- No silent crashes

---

## FOCUS MANAGEMENT

### Focus Operations

```javascript
// Add to ZuiRuntime object
const ZuiRuntime = {
    // ... existing fields ...

    // Focus management
    focusTrap: null,
    lastFocusedElement: null,

    // ... existing methods ...

    // ============== FOCUS MANAGEMENT ==============

    /**
     * Focus an element by ID
     */
    focusElement(elementId) {
        const element = this.getElement(elementId);
        if (!element) {
            console.warn(`Cannot focus element: ${elementId} not found`);
            return false;
        }

        // Make element focusable if not already
        if (!element.hasAttribute('tabindex') && !this.isFocusable(element)) {
            element.setAttribute('tabindex', '-1');
        }

        element.focus();
        return true;
    },

    /**
     * Focus element by selector
     */
    focusSelector(selector) {
        const element = document.querySelector(selector);
        if (!element) {
            console.warn(`Cannot focus selector: ${selector} not found`);
            return false;
        }

        if (!element.hasAttribute('tabindex') && !this.isFocusable(element)) {
            element.setAttribute('tabindex', '-1');
        }

        element.focus();
        return true;
    },

    /**
     * Check if element is naturally focusable
     */
    isFocusable(element) {
        const focusableTags = ['A', 'BUTTON', 'INPUT', 'TEXTAREA', 'SELECT'];
        return focusableTags.includes(element.tagName);
    },

    /**
     * Get currently focused element
     */
    getFocusedElement() {
        const active = document.activeElement;
        if (active === document.body) return null;
        return active;
    },

    /**
     * Save current focus
     */
    saveFocus() {
        this.lastFocusedElement = this.getFocusedElement();
    },

    /**
     * Restore previously saved focus
     */
    restoreFocus() {
        if (this.lastFocusedElement) {
            this.lastFocusedElement.focus();
            this.lastFocusedElement = null;
            return true;
        }
        return false;
    },

    /**
     * Move focus to next focusable element
     */
    focusNext() {
        const focusable = this.getFocusableElements();
        const current = this.getFocusedElement();
        const currentIndex = focusable.indexOf(current);

        if (currentIndex < focusable.length - 1) {
            focusable[currentIndex + 1].focus();
            return true;
        }
        return false;
    },

    /**
     * Move focus to previous focusable element
     */
    focusPrevious() {
        const focusable = this.getFocusableElements();
        const current = this.getFocusedElement();
        const currentIndex = focusable.indexOf(current);

        if (currentIndex > 0) {
            focusable[currentIndex - 1].focus();
            return true;
        }
        return false;
    },

    /**
     * Get all focusable elements
     */
    getFocusableElements(container = document.body) {
        const selector = 'a[href], button:not([disabled]), input:not([disabled]), ' +
                        'textarea:not([disabled]), select:not([disabled]), ' +
                        '[tabindex]:not([tabindex="-1"])';

        return Array.from(container.querySelectorAll(selector))
            .filter(el => {
                // Check visibility
                const style = window.getComputedStyle(el);
                return style.display !== 'none' &&
                       style.visibility !== 'hidden' &&
                       el.offsetParent !== null;
            });
    },

    /**
     * Install focus trap on container
     */
    installFocusTrap(containerId) {
        const container = this.getElement(containerId);
        if (!container) {
            console.warn(`Cannot install focus trap: container ${containerId} not found`);
            return false;
        }

        this.saveFocus();

        const focusTrap = {
            container,
            handleKeyDown: (e) => {
                if (e.key !== 'Tab') return;

                const focusable = this.getFocusableElements(container);
                if (focusable.length === 0) return;

                const first = focusable[0];
                const last = focusable[focusable.length - 1];

                if (e.shiftKey) {
                    // Shift + Tab: move backwards
                    if (document.activeElement === first) {
                        e.preventDefault();
                        last.focus();
                    }
                } else {
                    // Tab: move forwards
                    if (document.activeElement === last) {
                        e.preventDefault();
                        first.focus();
                    }
                }
            },
        };

        container.addEventListener('keydown', focusTrap.handleKeyDown);
        this.focusTrap = focusTrap;

        // Focus first element
        const focusable = this.getFocusableElements(container);
        if (focusable.length > 0) {
            focusable[0].focus();
        }

        return true;
    },

    /**
     * Remove focus trap
     */
    removeFocusTrap() {
        if (!this.focusTrap) return false;

        this.focusTrap.container.removeEventListener('keydown', this.focusTrap.handleKeyDown);
        this.focusTrap = null;

        this.restoreFocus();
        return true;
    },

    // ... rest of existing methods ...
};
```

### WASM Imports for Focus

```javascript
// Add to createImports() method
createImports() {
    return {
        zui: {
            // ... existing imports ...

            // Focus management
            js_focusElement: (elementId) => {
                return this.focusElement(elementId) ? 1 : 0;
            },

            js_focusSelector: (selectorPtr, selectorLen) => {
                const selector = this.ptrToStr(selectorPtr, selectorLen);
                return this.focusSelector(selector) ? 1 : 0;
            },

            js_saveFocus: () => {
                this.saveFocus();
            },

            js_restoreFocus: () => {
                return this.restoreFocus() ? 1 : 0;
            },

            js_focusNext: () => {
                return this.focusNext() ? 1 : 0;
            },

            js_focusPrevious: () => {
                return this.focusPrevious() ? 1 : 0;
            },

            js_installFocusTrap: (containerId) => {
                return this.installFocusTrap(containerId) ? 1 : 0;
            },

            js_removeFocusTrap: () => {
                return this.removeFocusTrap() ? 1 : 0;
            },
        },
    };
},
```

### Zig Integration

```zig
// src/effects/focus.zig
const std = @import("std");

// WASM imports for focus management
extern "zui" fn js_focusElement(element_id: u32) bool;
extern "zui" fn js_focusSelector(selector_ptr: [*]const u8, selector_len: usize) bool;
extern "zui" fn js_saveFocus() void;
extern "zui" fn js_restoreFocus() bool;
extern "zui" fn js_focusNext() bool;
extern "zui" fn js_focusPrevious() bool;
extern "zui" fn js_installFocusTrap(container_id: u32) bool;
extern "zui" fn js_removeFocusTrap() bool;

pub const Focus = struct {
    /// Focus an element by ID
    pub fn element(element_id: u32) void {
        _ = js_focusElement(element_id);
    }

    /// Focus element by CSS selector
    pub fn selector(sel: []const u8) void {
        _ = js_focusSelector(sel.ptr, sel.len);
    }

    /// Save current focus
    pub fn save() void {
        js_saveFocus();
    }

    /// Restore saved focus
    pub fn restore() bool {
        return js_restoreFocus();
    }

    /// Move to next focusable element
    pub fn next() bool {
        return js_focusNext();
    }

    /// Move to previous focusable element
    pub fn previous() bool {
        return js_focusPrevious();
    }

    /// Install focus trap on container
    pub fn trap(container_id: u32) bool {
        return js_installFocusTrap(container_id);
    }

    /// Remove focus trap
    pub fn untrap() bool {
        return js_removeFocusTrap();
    }
};

// Focus effect for Effects system
pub fn Effect(comptime Msg: type) type {
    return union(enum) {
        // ... existing effects ...

        /// Focus management effect
        focus: union(enum) {
            element: u32,
            selector: []const u8,
            save,
            restore,
            next,
            previous,
            trap: u32,
            untrap,
        },
    };
}
```

### Usage Examples

```zig
// Example 1: Focus element after mount
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .modal_opened => {
            // Focus first input in modal
            return Effect.focus(.{ .selector = "#modal input:first-child" });
        },

        // ... other cases
    };
}

// Example 2: Focus trap for modal
pub fn view(ctx: *AppContext, model: Model) Element(Msg) {
    return if (model.show_modal)
        h.div(ctx, Msg, .{
            .id = "modal-container",
            .class = "modal",
            .role = "dialog",
            .ariaModal = true,
        }, &.{
            h.h2(ctx, Msg, .{}, &.{h.text(ctx, Msg, "Modal Title")}),
            h.button(ctx, Msg, .{
                .onClick = .close_modal,
            }, &.{h.text(ctx, Msg, "Close")}),
        })
    else
        h.none(ctx, Msg);
}

pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .modal_opened => {
            model.show_modal = true;
            // Install focus trap
            const modal_id = 1234; // Element ID from registry
            return Effect.focus(.{ .trap = modal_id });
        },

        .close_modal => {
            model.show_modal = false;
            // Remove focus trap and restore focus
            return Effect.batch(Msg, &.{
                Effect.focus(.untrap),
                Effect.focus(.restore),
            });
        },

        // ... other cases
    };
}

// Example 3: Keyboard navigation
pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .key_down => |key| {
            if (std.mem.eql(u8, key, "ArrowDown")) {
                return Effect.focus(.next);
            } else if (std.mem.eql(u8, key, "ArrowUp")) {
                return Effect.focus(.previous);
            }
            return Effect.none(Msg);
        },

        // ... other cases
    };
}

// Example 4: Save/restore focus for dropdown
pub const Model = struct {
    dropdown_open: bool,
    dropdown_trigger_id: u32,
};

pub fn update(model: *Model, msg: Msg, ctx: *AppContext) Effect(Msg) {
    return switch (msg) {
        .toggle_dropdown => {
            if (model.dropdown_open) {
                // Closing - restore focus to trigger
                model.dropdown_open = false;
                return Effect.focus(.restore);
            } else {
                // Opening - save focus and focus first item
                model.dropdown_open = true;
                return Effect.batch(Msg, &.{
                    Effect.focus(.save),
                    Effect.focus(.{ .selector = ".dropdown-item:first-child" }),
                });
            }
        },

        // ... other cases
    };
}
```

---

## CONCLUSION

The enhanced JS runtime provides superior performance, safety, and developer experience. Focus management capabilities ensure accessible keyboard navigation and proper focus handling for interactive components like modals, dropdowns, and complex forms.

**Links:**
- [← Previous: WASM Runtime](13-wasm-runtime.md)
- [→ Next: Build System](15-build-system.md)
- [↑ Back to index](../README.md)
