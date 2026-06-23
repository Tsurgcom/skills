# Vite 8.1 WASM ESM — Reference

## TypeScript: `*.wasm` module declarations

Add to a `*.d.ts` file included by the app's tsconfig when importing known exports:

```ts
declare module "*.wasm" {
  export function add(a: number, b: number): number;
}
```

For `?init`:

```ts
declare module "*.wasm?init" {
  export default function init(
    imports?: WebAssembly.Imports,
  ): Promise<WebAssembly.Instance>;
}
```

Vite 8.1 also extends `vite/client` types for `.wasm` imports — ensure `vite/client` is referenced in `tsconfig.json` or `vite-env.d.ts`.

## WASM module with JS imports

`light-with-imports.wasm` pattern (from Vite playground):

```js
// imports.js — resolved relative to the .wasm file
export function getRandom() {
  return Math.random();
}
```

```js
// consumer.js
import { getRandomNumber } from "./light-with-imports.wasm";
```

Vite wires `getRandom` from `imports.js` into the WASM import object.

## Async module patterns

Direct `.wasm` imports are async modules. Valid patterns:

```ts
// Top-level await (module context)
import { add } from "./add.wasm";
const result = add(1, 2);

// Dynamic import from a function
async function run() {
  const { add } = await import("./add.wasm");
  return add(1, 2);
}

// Lazy / client-only
const mod = await import("./engine.wasm");
```

## Chunk import map — cache invalidation diagram

Without import map:

```
utils.[hash-A].js  (content changes → hash-B)
  ↑ imported by
page.[hash-C].js   (import URL embeds hash-A → re-hashed to hash-D)
  ↑ imported by
entry.[hash-E].js  (cascade continues)
```

With `build.chunkImportMap: true`:

```
import map: { "chunk-utils": "/assets/utils.[hash-B].js" }
page.js imports "chunk-utils" by ID → page.js hash unchanged
```

Only `utils.[hash-B].js` and the import map entry need re-fetch after a utils change.

## vite.config.ts snippets

### WASM + chunk import map

```ts
import { defineConfig } from "vite";

export default defineConfig({
  build: {
    chunkImportMap: true,
    // assetsInlineLimit: 4096, // default; affects small .wasm inlining
  },
});
```

### Relative `base` (Tauri, Electron, file://)

Desktop shells often require relative asset URLs. WASM fetches must resolve under the packaged protocol:

```ts
export default defineConfig({
  base: "./",
  build: {
    modulePreload: { polyfill: false },
  },
});
```

After enabling native `.wasm` imports, verify packaged builds load `.wasm` files from the shell's asset protocol. Fetch failures usually mean a wrong `base` or absolute URL in the built output.

## wasm-pack output structure (`--target web`)

```
dist/
  index.js          # JS glue, default export = init
  index_bg.wasm     # binary
  index.d.ts        # TypeScript types
```

Typical glue init:

```ts
import init, { MyApi } from "my-wasm-pkg";

await init(); // fetch + instantiate index_bg.wasm
const api = new MyApi();
```

Vite's native WASM ESM would instead look like:

```ts
import { some_export } from "./index_bg.wasm";
```

wasm-bindgen does not emit WASM/ESM-compatible export layouts today; the `--target web` glue remains the correct integration for `#[wasm_bindgen]` crates.

## Migrating from vite-plugin-wasm

1. Remove `vite-plugin-wasm` and `vite-plugin-top-level-await` from `vite.config.ts` (if only used for WASM).
2. Replace `import init from "./foo.wasm"` (plugin style) with either:
   - `import { fn } from "./foo.wasm"` (direct), or
   - `import init from "./foo.wasm?init"` (manual).
3. Update TypeScript declarations.
4. Run dev + production builds; inspect network tab for `.wasm` fetch vs inline.

## Debugging

| Symptom | Likely cause |
|---------|----------------|
| `Failed to fetch` for `.wasm` | Wrong `base` path (`./` vs `/`) or broken asset URL in production |
| Top-level await error | Target too old; use dynamic `import()` or raise `build.target` |
| Empty exports | Wrong module declaration; binary may not export expected names |
| SSR crash on `.wasm` import | Feature requires Node.js runtime with `node:fs` |
| Chunk import map blank in old browser | Missing `import.meta.resolve`; use modern target or disable feature |
| `renderBuiltUrl` ignored | Incompatible with `chunkImportMap` |

## Links

- [Vite 8.1 blog post](https://vite.dev/blog/announcing-vite8-1)
- [WebAssembly features guide](https://vite.dev/guide/features#webassembly)
- [Chunk import map optimization](https://vite.dev/guide/features#chunk-import-map-optimization)
- [WASM ESM integration proposal](https://github.com/WebAssembly/esm-integration)
- [Chunk import map PR #21580](https://github.com/vitejs/vite/pull/21580)
- [WASM ESM PR #21779](https://github.com/vitejs/vite/pull/21779)
