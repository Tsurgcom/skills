---
name: vite-8-wasm-esm
description: Work with Vite 8.1+ WASM ESM integration, chunk import map optimization, and related experimental build features. Use when importing .wasm files in Vite, migrating from vite-plugin-wasm, configuring build.chunkImportMap, or evaluating wasm-pack/wasm-bindgen vs native WASM ESM.
---

# Vite 8.1 WASM ESM & Build Optimizations

Vite 8.1 upstreamed WASM ESM integration (previously `vite-plugin-wasm`) and added experimental chunk import map optimization. Both are bleeding-edge — verify behavior in dev and production builds before relying on them.

**References:** [Vite 8.1 announcement](https://vite.dev/blog/announcing-vite8-1) · [Features: WebAssembly](https://vite.dev/guide/features#webassembly) · [Features: Chunk Import Map](https://vite.dev/guide/features#chunk-import-map-optimization)

## Version gate

Require **Vite ≥ 8.1** for these features. Check the consuming app's `vite` version before applying patterns.

```bash
npx vite --version
# or: pnpm exec vite --version
```

## WASM ESM integration

Vite supports two import styles for pre-compiled `.wasm` files.

### Direct import (preferred when exports are enough)

```ts
import { add } from "./add.wasm";

console.log(add(1, 2)); // 3
```

- Vite parses the binary, instantiates the module, and re-exports WASM exports as named ES exports.
- Follows the [WebAssembly/ES Module Integration](https://github.com/WebAssembly/esm-integration) proposal.
- **Async module:** direct `.wasm` imports are async modules; callers need top-level `await` or dynamic `import()`.
- **WASM imports:** if the `.wasm` declares imports, Vite resolves each import name as a JS module specifier (relative to the `.wasm` file) and wires members automatically.

### Manual init (`?init`)

Use when you need control over instantiation timing or an `importObject`:

```ts
import init from "./example.wasm?init";

const instance = await init({
  imports: { env: { someFunc: () => {} } },
});
instance.exports.test();
```

### URL import (multiple instances / raw Module)

```ts
import wasmUrl from "./foo.wasm?url";

const { module, instance } = await WebAssembly.instantiateStreaming(fetch(wasmUrl));
```

### Production behavior

- `.wasm` smaller than `build.assetsInlineLimit` → inlined as base64.
- Larger files → emitted as static assets, fetched on demand.

### SSR caveat

Direct `.wasm` and `.wasm?init` use `node:fs` internally. **SSR builds only work in Node.js-compatible runtimes.**

### Remove `vite-plugin-wasm` when migrating

If the project only used `vite-plugin-wasm` for basic `.wasm` imports, remove the plugin and use native imports. Keep the plugin only if you depend on behavior Vite core does not replicate.

## Chunk import map optimization

Experimental in 8.1. Improves long-term cache hit rate by decoupling chunk hashes from import statements.

**Problem:** static `import` URLs embed the imported chunk's hash. When chunk C changes, every importer (B, A, entry) is re-hashed even though only C's content changed.

**Solution:** Vite emits an import map mapping chunk IDs → hashed URLs; import statements reference IDs instead.

### Enable

```ts
// vite.config.ts
export default defineConfig({
  build: {
    chunkImportMap: true,
  },
});
```

### Constraints (read before enabling)

| Constraint | Detail |
|------------|--------|
| Browser support | Requires native `import.meta.resolve` |
| `experimental.renderBuiltUrl` | **Incompatible** — do not combine |
| CSS / assets | Optimization does not apply; asset changes still invalidate direct importers (no cascade beyond that) |
| `@vitejs/plugin-legacy` | Legacy path uses SystemJS import maps (8.1+) |

Gather feedback and test cache behavior in staging before shipping to production.

## wasm-pack / wasm-bindgen vs Vite WASM ESM

These solve different layers:

| Layer | What it does |
|-------|----------------|
| **wasm-pack `--target web`** | Generates wasm-bindgen JS glue (`__wbindgen_*`, `default()` init, typed exports) alongside `*_bg.wasm` |
| **Vite WASM ESM** | Treats a `.wasm` binary as an ES module per the WASM/ESM proposal — no wasm-bindgen glue |

**Do not** replace a wasm-bindgen package with a raw `.wasm` import unless you also replace the bindings layer. A migration means rethinking how exported classes and functions are exposed to TypeScript.

**Use Vite WASM ESM when:**

- Importing standalone `.wasm` binaries (e.g. compiled from WAT, Emscripten without JS glue, or other toolchains)
- Prototyping small WASM modules without a full wasm-pack crate
- The module's exports map cleanly to the WASM/ESM proposal

**Keep wasm-pack / wasm-bindgen when:**

- Rust `#[wasm_bindgen]` APIs are consumed from TypeScript
- The crate uses `web-sys`, `js-sys`, or other bindgen-generated bindings
- An existing singleton init pattern wraps `import("pkg")` then `mod.default()`

Typical wasm-pack init (unchanged by Vite 8.1):

```ts
import init, { MyApi } from "my-wasm-pkg";

await init();
const api = new MyApi();
```

## Workflow checklist

When adding or changing WASM loading:

```
- [ ] Confirm Vite ≥ 8.1
- [ ] Choose import style: direct | ?init | ?url | wasm-pack package
- [ ] Verify dev server (vite dev)
- [ ] Verify production build (vite build) — check .wasm asset paths and base URL
- [ ] If enabling chunkImportMap: test in target browsers; confirm no renderBuiltUrl usage
- [ ] If using wasm-pack: rebuild WASM artifacts before the Vite build
- [ ] Run typecheck after TS changes
```

## Related Vite 8.1 experimental features

Only enable when explicitly asked — not required for WASM ESM.

| Feature | Config | Notes |
|---------|--------|-------|
| Bundled dev | `experimental.bundledDev: true` or `--experimental-bundle` | Bundles in dev for large apps; third-party plugins may break |
| Lightning CSS default path | `css.transformer: 'lightningcss'` | Future default; test PostCSS parity |

## Examples

Short patterns above; extended copy-paste examples in [examples.md](examples.md):

- Direct import, `?init`, `?url` in React routes and workers
- Production inlining vs separate `.wasm` assets; relative `base` for desktop shells
- Chunk import map with manual chunks
- wasm-pack singleton loader; composing sidecar `.wasm` modules
- SSR, `import.meta.glob`, and `vite-plugin-wasm` migration before/after

## Additional resources

- API notes, type shims, debugging: [reference.md](reference.md)
- Full worked examples: [examples.md](examples.md)
