# Vite 8.1 WASM ESM — Examples

Concrete patterns for dev, production, React, SSR, workers, and wasm-pack coexistence.

## 1. Direct `.wasm` import

### Minimal arithmetic module

```ts
// src/math.ts
import { add, mul } from "../assets/add.wasm";

export function dot2(a: [number, number], b: [number, number]) {
  return add(mul(a[0], b[0]), mul(a[1], b[1]));
}
```

```ts
// src/vite-env.d.ts (or wasm.d.ts)
declare module "*.wasm" {
  export function add(a: number, b: number): number;
  export function mul(a: number, b: number): number;
}
```

### Dynamic import (no top-level await required)

```ts
export async function runWasmBenchmark() {
  const { add } = await import("../assets/add.wasm");
  const start = performance.now();
  let sum = 0;
  for (let i = 0; i < 1_000_000; i++) sum = add(sum, 1);
  return { sum, ms: performance.now() - start };
}
```

### React: load WASM only on a heavy route

```tsx
// src/routes/editor.tsx
import { useEffect, useState } from "react";

type WasmExports = typeof import("../wasm/filters.wasm");

export function EditorPage() {
  const [wasm, setWasm] = useState<WasmExports | null>(null);

  useEffect(() => {
    let cancelled = false;
    import("../wasm/filters.wasm").then((mod) => {
      if (!cancelled) setWasm(mod);
    });
    return () => {
      cancelled = true;
    };
  }, []);

  if (!wasm) return <p>Loading engine…</p>;
  return <Canvas apply={wasm.applyBlur} />;
}
```

### Code-split WASM alongside a JS chunk

```ts
// vite produces a separate chunk for the .wasm import
async function openTimeline() {
  const [{ mountTimeline }, wasm] = await Promise.all([
    import("./timeline-ui"),
    import("./timeline-snap.wasm"),
  ]);
  mountTimeline({ snap: wasm.snapToFrame });
}
```

---

## 2. `?init` — manual instantiation

### Defer init until user gesture (autoplay / permission patterns)

```ts
import init from "./audio-proc.wasm?init";

let instance: WebAssembly.Instance | null = null;

export async function startAudioPipeline() {
  if (!instance) {
    instance = await init();
  }
  (instance.exports.process as (ptr: number) => void)(bufferPtr);
}
```

### Custom import object (WASM imports from JS)

```ts
import init from "./light-with-imports.wasm?init";

const instance = await init({
  "./imports.js": {
    getRandom: () => Math.random(),
    now: () => performance.now(),
  },
});

const n = (instance.exports.getRandomNumber as () => number)();
```

With Vite 8.1 direct import, the same WASM module can omit the manual `importObject` when imports are real JS modules:

```ts
// src/wasm/imports.ts
export function getRandom() {
  return Math.random();
}
```

```ts
// src/wasm/consumer.ts — Vite resolves "./imports.ts" from the .wasm binary's imports
import { getRandomNumber } from "./light-with-imports.wasm";
```

### Merge Vite-resolved imports with overrides

```ts
import init from "./plugin.wasm?init";

const instance = await init({
  env: {
    log: (ptr: number, len: number) => {
      console.log(readWasmString(ptr, len));
    },
  },
});
```

---

## 3. `?url` — fetch, cache, reinstantiate

### Singleton + warm-up

```ts
import wasmUrl from "./heavy.wasm?url";

let compiled: WebAssembly.Module | null = null;

export async function getCompiledModule() {
  if (!compiled) {
    const res = await fetch(wasmUrl);
    compiled = await WebAssembly.compileStreaming(res);
  }
  return compiled;
}

export async function createInstance(imports: WebAssembly.Imports) {
  const module = await getCompiledModule();
  return WebAssembly.instantiate(module, imports);
}
```

### Two instances from one binary (e.g. A/B workers)

```ts
import wasmUrl from "./worker-kernel.wasm?url";

async function spawnKernel(id: number) {
  const { instance } = await WebAssembly.instantiateStreaming(
    fetch(wasmUrl),
    { env: { id: () => id } },
  );
  return instance;
}

const [a, b] = await Promise.all([spawnKernel(1), spawnKernel(2)]);
```

---

## 4. Production build output

### Small WASM inlined (< `assetsInlineLimit`, default 4096 bytes)

```ts
// vite.config.ts
export default defineConfig({
  build: {
    assetsInlineLimit: 8192, // raise to inline slightly larger modules
  },
});
```

Network tab in production: no separate `.wasm` request — glue embeds base64.

### Large WASM as static asset

```ts
// vite.config.ts
export default defineConfig({
  build: {
    assetsInlineLimit: 0, // force separate .wasm files for all sizes
  },
});
```

`dist/assets/heavy-a1b2c3d4.wasm` is fetched at runtime; hash in filename enables long-term caching.

### Relative `base` (Tauri, Electron, subpath deploy)

```ts
// vite.config.ts
export default defineConfig({
  base: "./",
});
```

```ts
// Built chunk (conceptual) — URL resolves relative to the JS file
const wasmUrl = new URL("./assets/kernel-a1b2c3d4.wasm", import.meta.url);
```

Verify in packaged or subpath builds: the shell's asset protocol or CDN prefix must resolve the `.wasm` URL. If fetch fails, check `base` and built output paths.

---

## 5. Chunk import map

### Enable and inspect output

```ts
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  build: {
    chunkImportMap: true,
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes("node_modules")) return "vendor";
          if (id.includes("/src/lib/")) return "lib";
        },
      },
    },
  },
});
```

Conceptual built HTML:

```html
<script type="importmap">
{
  "imports": {
    "chunk-vendor": "/assets/vendor.f8e2.js",
    "chunk-lib": "/assets/lib.c4a1.js"
  }
}
</script>
<script type="module" src="/assets/index.d39b.js"></script>
```

`index.js` contains `import "chunk-lib"` (ID) instead of `import "/assets/lib.c4a1.js"`. When only `lib` changes, `vendor` and `index` hashes stay stable.

### With route-level code splitting

Framework routers (React Router, TanStack Router, etc.) already split by route. `chunkImportMap` helps when shared vendor/lib chunks change less often than route chunks:

```
After deploy, user revisits /dashboard:
  Without import map → entry + transitive chunks may all re-download
  With import map     → only changed route chunk + import map patch
```

---

## 6. wasm-pack / wasm-bindgen (coexists with Vite WASM ESM)

Vite 8.1 does not replace wasm-pack. Use the generated JS package for `#[wasm_bindgen]` APIs; use native WASM ESM for standalone binaries.

### Singleton loader

```ts
// src/lib/wasm.ts
let wasmPromise: Promise<typeof import("my-wasm-pkg")> | null = null;

export async function loadWasm() {
  if (typeof window === "undefined") {
    throw new Error("WASM is only available in the browser");
  }
  if (!wasmPromise) {
    wasmPromise = import("my-wasm-pkg").then(async (mod) => {
      await mod.default(); // fetches *_bg.wasm, runs __wbindgen_start
      return mod;
    });
  }
  return wasmPromise;
}
```

### Bootstrap alongside browser APIs

```ts
import { loadWasm } from "./lib/wasm";
import { initCanvas } from "./lib/canvas";

export async function bootApp(canvas: HTMLCanvasElement) {
  const [{ Renderer }, ctx] = await Promise.all([
    loadWasm(),
    initCanvas(canvas),
  ]);
  Renderer.draw(ctx);
  return ctx;
}
```

### Rebuild WASM before Vite build

```bash
# in the wasm package
wasm-pack build --target web

# in the Vite app
vite build
```

---

## 7. Composing wasm-pack with native WASM ESM

Use native `.wasm` imports only for **non–wasm-bindgen** binaries. Do not import a wasm-pack `*_bg.wasm` directly — it expects bindgen glue.

### Sidecar module (e.g. `wat2wasm` LUT filter)

```ts
// src/lib/lut.ts
export async function loadLut() {
  const { applyLut } = await import("../assets/color-lut.wasm");
  return applyLut;
}
```

```ts
// Compose with wasm-bindgen engine
const { Engine } = await loadWasm();
const applyLut = await loadLut();
// Engine handles main API; applyLut is a separate WASM module
```

### Import package `.wasm` via alias

```ts
// vite.config.ts
import path from "node:path";

export default defineConfig({
  resolve: {
    alias: {
      "@wasm-assets": path.resolve(__dirname, "../wasm-assets/dist"),
    },
  },
});
```

```ts
import { hash } from "@wasm-assets/fingerprint.wasm";
```

---

## 8. SSR (Node.js only)

```ts
// src/entry-server.tsx — only in Node-compatible SSR
import { add } from "./add.wasm";

export function prerenderStats() {
  return { checksum: add(1, 2) };
}
```

```ts
// src/entry-client.tsx — browser path unchanged
const { add } = await import("./add.wasm");
```

SSR build uses `node:fs` to read `.wasm` from disk. Edge runtimes without Node `fs` are unsupported for direct `.wasm` imports.

---

## 9. Web Worker + WASM

### Worker imports WASM directly (module worker)

```ts
// src/workers/decode.worker.ts
import { decodeFrame } from "../wasm/decode.wasm";

self.onmessage = (e: MessageEvent<ArrayBuffer>) => {
  const out = decodeFrame(new Uint8Array(e.data));
  self.postMessage(out, [out.buffer]);
};
```

```ts
// src/lib/decode-pool.ts
const worker = new Worker(
  new URL("../workers/decode.worker.ts", import.meta.url),
  { type: "module" },
);
```

### `?init` inside worker for explicit imports

```ts
// src/workers/proc.worker.ts
import init from "../wasm/proc.wasm?init";

const instance = await init();
self.onmessage = () => {
  (instance.exports.run as () => void)();
};
```

---

## 10. `import.meta.glob` for multiple WASM modules

```ts
const modules = import.meta.glob("../filters/*.wasm");

export async function loadFilter(name: string) {
  const loader = modules[`../filters/${name}.wasm`];
  if (!loader) throw new Error(`Unknown filter: ${name}`);
  return loader() as Promise<typeof import("../filters/blur.wasm")>;
}

// Usage
const blur = await loadFilter("blur");
blur.apply(srcPtr, dstPtr, radius);
```

---

## 11. Migrate from `vite-plugin-wasm`

### Before (Vite 5 + plugins)

```ts
// vite.config.ts
import wasm from "vite-plugin-wasm";
import topLevelAwait from "vite-plugin-top-level-await";

export default defineConfig({
  plugins: [wasm(), topLevelAwait()],
});
```

```ts
// app.ts
import init from "./add.wasm";

await init();
```

### After (Vite 8.1 native)

```ts
// vite.config.ts — remove both plugins
export default defineConfig({
  plugins: [],
});
```

```ts
// Option A: direct (simplest)
import { add } from "./add.wasm";
console.log(add(1, 2));

// Option B: ?init (closest to old plugin default export)
import init from "./add.wasm?init";
const instance = await init();
```

---

## 12. `vite.config.ts` combinations

### WASM-heavy app + chunk import map

```ts
import { defineConfig } from "vite";

export default defineConfig({
  build: {
    target: "esnext", // async modules / import.meta.resolve
    chunkImportMap: true,
    assetsInlineLimit: 0, // always emit .wasm as files (easier to debug)
  },
  worker: {
    format: "es",
  },
});
```

### Web vs desktop shell

```ts
const isDesktop = process.env.BUILD_TARGET === "desktop";

export default defineConfig({
  base: isDesktop ? "./" : "/",
  build: {
    chunkImportMap: !isDesktop, // validate desktop asset protocol separately
    ...(isDesktop && { modulePreload: { polyfill: false } }),
  },
});
```

### Force separate WASM files in production

```ts
export default defineConfig({
  build: {
    assetsInlineLimit(file) {
      if (file.endsWith(".wasm")) return false;
      return undefined; // default for other assets
    },
  },
});
```

---

## 13. Quick decision guide

| Goal | Pattern |
|------|---------|
| Call exported functions, fire-and-forget | `import { fn } from "./x.wasm"` |
| Init on click / lazy route | `await import("./x.wasm")` |
| Custom `importObject` / legacy init | `import init from "./x.wasm?init"` |
| Compile once, instantiate many | `import url from "./x.wasm?url"` |
| `#[wasm_bindgen]` Rust API | `import("my-wasm-pkg")` + `default()` |
| Better chunk cache on deploy | `build.chunkImportMap: true` |
| SSR prerender with WASM | direct import in Node SSR entry only |
