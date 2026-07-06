# kit.js — Ultra-lightweight client-side interactive runtime extracted from Kitwork Engine

**kit.js** is not a traditional JavaScript framework. It is an ultra-lightweight interactive client runtime (~27KB minified, ~7KB gzipped) born and extracted directly from the core design philosophy of **Kitwork Engine**.

The mission of `kit.js` is to deliver powerful reactive state management and a seamless Single Page Application (SPA) experience directly on plain HTML markup **without build steps, without `node_modules`, without a Virtual DOM, and absolutely without ever using `eval()`**.

```html
<!-- Include the kit.js interactive kernel -->
<script src="https://cdn.jsdelivr.net/npm/@kitwork/kitjs@1"></script>

<!-- Define the interactive application boundary -->
<section data-kit-app="runtime@v1.0.0">
  <button data-kit-click="n = n + 1">+1</button>
  <b data-kit-text="n * qty">0</b>
  <input type="number" data-kit-model="qty" value="2">
  <span data-kit-show="n > 3">Unlocked</span>
</section>
```

All interactions, state modifications, and event bindings are declared inline via standard HTML attributes (`data-kit-*`). Anyone inspecting the page's source code can instantly read and understand exactly how every element behaves.

---

## 1. Relationship with Kitwork Engine

`kit.js` is the standalone client-side build of **jitjs** — the JIT frontend delivery module of **Kitwork Engine** (written in Go). When running within the Kitwork ecosystem:

* **Server-Side JIT Verification:** Every expression declared in `data-kit-*` attributes is pre-compiled and syntax-verified by the Go engine at render time. Typos are caught on the server before the page reaches the user.
* **One Rule, Two Ends Validation:** A validation expression in `data-kit-validate` runs reactively on the client as you type, and the *exact same* compiled expression is validated on the Go server upon form submission (`ctx.validate`), eliminating client-server validation drift.
* **Zero-Toolchain Frontend:** Combined with Kitwork Engine, all stylesheets, icons, fonts, and client-side behaviors are compiled JIT and streamed on demand via `jitcss`, `jiticons`, `jitfonts`, and `jitjs`.

---

## 2. Standalone Capabilities of kit.js (Any Backend)

Even when running standalone without the Go backend engine, `kit.js` works out-of-the-box, providing:

| Capability | Declarative Directive | Description |
| :--- | :--- | :--- |
| **Expressions** | `data-kit-text="n * qty"` | Compiled into a lightweight AST representation and executed via a secure AST walker, fully complying with strict Content Security Policies (CSP). |
| **Event Delegation** | `data-kit-click="n = n + 1"` | A single delegated event listener bound to `document` handles all events, reducing memory footprint and avoiding leaks. |
| **Two-Way Binding** | `data-kit-model="qty"` | Automatically synchronizes element values with local scope, performing type coercion (e.g. converting to a `number` when `type="number"`). |
| **Validation UX** | `data-kit-validate="confirm == password"` | Evaluates inputs on the fly, applying `data-state="valid\|invalid"` for easy CSS styling and preventing invalid form submissions. |
| **Realtime SSE** | `data-kit-live="/api/sse"` | Automatically establishes an SSE connection, merging incoming JSON patches into the page state. Auto-closes when the DOM element is removed to save resources. |
| **SPA Navigation** | Declared on `data-kit-app` | Automatically transforms relative links and forms into AJAX requests. Fetches new content and performs smooth DOM **morphing** (preserving input focus, scroll position, and cursor states). |
| **Custom Verbs** | `data-kit-action="copy"` | Easily extend kernel behaviors by registering custom JavaScript actions via `kitwork.behavior()`. |

---

## 3. Attribute Reference Guide

You can use the shorthand `data-kit-*` or the canonical `data-kitwork-*` spelling interchangeably.

| Attribute | Data Type | Description |
| :--- | :--- | :--- |
| `data-kit-app` | `"[mode]@[version]"` or `false` | Initializes the local runtime scope and enables SPA navigation. |
| `data-kit-text` | JS Expression | Computes the expression and assigns the result to the element's `textContent`. |
| `data-kit-show` | JS Expression | Shows or hides the element based on the truthiness of the expression. |
| `data-kit-click` | JS Expression | Executes the expression whenever the element is clicked. |
| `data-kit-model` | State Key (string) | Synchronizes scope state in a two-way fashion with input elements. |
| `data-kit-validate` | Logic Expression | Triggers state validation on inputs, applying `data-state` to elements. |
| `data-kit-live` | SSE Endpoint URL | Streams real-time updates and merges JSON patches into the scope state. |
| `data-kit-scope` | Scope Name | Creates a local state boundary (local scope) supporting nested prototype inheritance. |
| `data-kit-remember` | Space-separated keys | Persists and automatically restores specified scope state variables from `localStorage`. |
| `data-kit-indexed` | `"true" \| "false"` | Enables local IndexedDB caching and offline persistence for dynamic content. |
| `data-kit-api` | JSON API Endpoint | Fetches remote JSON data on mount, providing state indicators (`loading`, `ready`, `error`). |
| `data-kit-trigger` | `"visible"` | Triggers the registered action as soon as the element enters the viewport. |
| `data-kit-component` | Component Name | Hydrates the element template with the specified registered component layout. Supports aliasing via `=`. |
| `data-kit-alias` | Alias Name (e.g. `$name`) | Registers a global alias reference to control this element/scope from anywhere. |
| `data-kit-action` | Action Name | Binds a custom behavior verb to this element. |

---

## 4. State Scopes & Imperative Escape Hatches

### A. Hierarchical State Management (Scopes)
State is global to the page unless bounded by a `data-kit-scope` directive. When a scope is declared:
* Read operations bubble upward, retrieving values from ancestor scopes if not present in the local boundary.
* Write operations (variable assignments) are bound directly to the local scope.
* Use the special `$.` prefix to read or write directly to the page's root scope (the same `$` context used by server-side templates).

```html
<div data-kit-scope="counter">
  <!-- These two counters run completely independently, each holding its own 'n' -->
  <button data-kit-click="n = n + 1">+</button> <b data-kit-text="n">0</b>
</div>
<div data-kit-scope="counter">
  <button data-kit-click="n = n + 1">+</button> <b data-kit-text="n">0</b>
</div>
```

### B. Imperative Escape Hatches: `$el`, `$root`, `$app`, and Aliases
For situations requiring imperative DOM actions (such as focusing an input, scrolling elements, or interfacing with third-party libraries), `kit.js` exposes special reference keywords:
* `$el` — Resolves to the current native DOM element where the directive is placed.
* `$root` — Resolves to the nearest parent DOM element defining `data-kit-scope` (or `<html>` if none exists).
* `$app` — The native bridge portal exposing hardware functions and OS-level APIs in expressions:
  * `$app.camera(scopeKey)`: Activates the hardware camera to take a photo, binding the file cache URI directly to the specified `scopeKey`.
  * `$app.qrcode(scopeKey)`: Starts the native QR code reader, resolving the scanned token back to the `scopeKey`.
  * `$app.biometrics(scopeKey)`: Triggers system biometric verification (FaceID / Fingerprint).
* **Aliases (e.g. `$sider`)** — Custom aliases also act as direct imperative escape hatch references. Any expression on the page can use the alias directly to interact with that element's API or scope.

```html
<!-- Take a photo via the native hardware bridge -->
<button data-kit-click="$app.camera('user_avatar')">Capture Avatar</button>
```

#### Component & Element Aliasing Mechanism
To easily control or trigger methods on component boundaries from other unrelated parts of the DOM, you can assign an alias starting with a `$` prefix:
1. **Component Aliases:** Declared directly in the component instantiation by appending `=[alias_name]` after the template name:
   * `data-kit-component="sidebar=$sider"`
   * `data-kit-component="sidebar@1.0.0=$sider"`
2. **Element/Scope Aliases:** Declared using the `data-kit-alias` attribute on standard elements:
   * `<div data-kit-alias="$sider" data-kit-scope="navigation">...</div>`

Once registered, you can invoke methods and control its state from anywhere on the page:

```html
<!-- Triggers the sider's open action from an unrelated button -->
<button data-kit-click="$sider.open()">Open Sidebar</button>
```

---

## 5. Future Development Roadmap

`kit.js` remains committed to its core philosophy of ultra-lightweight declarative markup while introducing deep hardware interfaces and offline performance optimizations:

### A. Declarative Hardware Bridge
Continue expanding system-level integration via the `$app` global portal, supporting automated environment detection (standard browser behaviors vs. Go core bridge app environments):
* **`$app.camera(scopeKey)`**: Smooth native overlays for iOS/Android/Desktop cameras with HTML5 `<input type="file" capture="user">` browser fallbacks.
* **`$app.qrcode(scopeKey)`**: High-performance camera QR scanner overlays with canvas-based WebRTC browser fallbacks.
* **`$app.biometrics(scopeKey)`**: Unified OS biometric APIs.

### B. High-Performance DOM Morphing (Morphing Engine v2)
* Further optimization of DOM diffing algorithms to handle virtualized rendering of extremely large list elements.
* Native transition and layout animations triggered automatically when DOM nodes are inserted, removed, or rearranged.

### C. Offline-First PWA Architecture
* Extensions for `data-kit-indexed` to automatically synchronize and persist client state to IndexedDB transparently.
* Out-of-the-box Service Worker integrations to turn any kit.js page into an offline-capable Progressive Web App with zero configurations.

### D. JIT Component Directory
* Automated dynamic importing of custom components from local paths or CDN directories upon detecting dynamic component tags inside the DOM, keeping the initial HTML payload as lightweight as possible.

---

## 6. Contributing

* **Closed Grammar, Open Registry:** We strictly decline pull requests proposing changes or additions to the core expression language syntax. All new features and customizations must be added via custom registered behaviors (`kitwork.behavior(...)`). This policy guarantees the kernel remains small, maintainable, and readable forever.
* **Single Source of Truth:** All files in the `dist/` directory are auto-generated from the Kitwork Go engine workspace. Please refer to [BUILDING.md](BUILDING.md) to rebuild the dist files, and do not modify files in `dist/` manually.

---

## 7. License

Released under the [MIT](LICENSE) License © Huỳnh Nhân Quốc — Kitwork
