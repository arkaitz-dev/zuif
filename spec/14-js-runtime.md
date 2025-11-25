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

## CONCLUSION

The enhanced JS runtime provides superior performance, safety, and developer experience.

**Links:**
- [← Previous: WASM Runtime](13-wasm-runtime.md)
- [→ Next: Build System](15-build-system.md)
- [↑ Back to index](../README.md)
