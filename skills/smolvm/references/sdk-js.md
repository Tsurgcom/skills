# smolvm JavaScript / Node.js SDK Reference

> **Source**: `lib/js/` in the smol-machines/smolvm GitHub repository.
> This reference is derived from the repo README and examples. The `lib/js/` directory
> is the canonical source — always check there for the most current API.

## Overview

The JS SDK lets you spawn and control microVMs programmatically from a Node.js application.
It wraps the same underlying libkrun/crun stack as the CLI.

Use cases:
- Embed a sandboxed execution environment in a Node.js server or tool
- Run untrusted agent-generated code in isolation
- Build CI pipelines that spin up ephemeral VMs without Docker

## Installation

The JS library lives in `lib/js/` and is intended to be used directly or vendored.
Check the repo for any published npm package names — as of April 2026 the library
is still in early development alongside the CLI.

```bash
# From the repo root — link or copy lib/js into your project
cd lib/js
npm install   # installs any dependencies
```

## Quick Start

```js
import { SmolVM } from './lib/js/index.js';  // or wherever you've placed the lib

// Ephemeral VM — auto-cleaned up when the block ends
const vm = new SmolVM({ image: 'alpine:latest', net: true });
const result = await vm.run('echo hello from a microvm');
console.log(result.stdout);   // "hello from a microvm"
await vm.destroy();
```

## API Reference

The following documents the JS API as exposed by `lib/js/`. For any surface not
listed here, inspect `lib/js/` directly in the repo.

---

### `new SmolVM(options)`

Create a new VM instance (does not start it yet).

```js
const vm = new SmolVM({
  image: 'python:3.12-alpine',  // container image
  net: true,                    // enable outbound TCP/UDP networking
  allowHosts: ['api.stripe.com'],  // egress allowlist (requires net: true)
  volumes: ['/tmp:/workspace'], // HOST:GUEST directory mounts
  env: { MY_KEY: 'my_value' }, // environment variables
  sshAgent: false,             // forward host SSH agent
  cpus: 4,                     // vCPU count (default: 4)
  mem: 8192,                   // RAM in MiB (default: 8192)
});
```

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `image` | string | required | OCI container image |
| `net` | boolean | `false` | Enable outbound networking |
| `allowHosts` | string[] | `[]` | Allowlist for egress (requires `net: true`) |
| `volumes` | string[] | `[]` | `HOST:GUEST` directory mounts (directories only) |
| `env` | object | `{}` | Environment variables injected into the guest |
| `sshAgent` | boolean | `false` | Forward host SSH agent |
| `cpus` | number | `4` | Number of vCPUs |
| `mem` | number | `8192` | RAM in MiB |

---

### `vm.run(command, options?)`

Run a shell command inside an **ephemeral** VM (the VM is created, command runs, VM is
destroyed). Returns a `Promise<RunResult>`.

```js
const result = await vm.run('python3 -c "import sys; print(sys.version)"');
console.log(result.stdout);
console.log(result.stderr);
console.log(result.exitCode);  // 0 on success
```

**`RunResult`:**

```ts
interface RunResult {
  stdout: string;
  stderr: string;
  exitCode: number;
}
```

---

### `vm.start()`

Start a **persistent** named VM. The VM continues running until explicitly stopped.

```js
await vm.start();
```

---

### `vm.exec(command, options?)`

Run a command inside an **already-running** persistent VM. Returns `Promise<RunResult>`.

```js
await vm.start();
await vm.exec('apk add git');
const result = await vm.exec('git --version');
console.log(result.stdout);
```

---

### `vm.stop()`

Stop a running persistent VM.

```js
await vm.stop();
```

---

### `vm.destroy()`

Stop and clean up an ephemeral VM (also callable on persistent VMs to fully remove them).

```js
await vm.destroy();
```

---

### `SmolVM.pack(options)` — build portable executables

```js
import { SmolVM } from './lib/js/index.js';

await SmolVM.pack({
  image: 'python:3.12-alpine',
  output: './my-pythonvm',
  singleFile: false,  // if true: single binary, no .smolmachine sidecar
});
// Produces: ./my-pythonvm + ./my-pythonvm.smolmachine
```

---

## Usage Patterns

### Sandboxed code execution for agents

```js
async function runUntrustedCode(code) {
  const vm = new SmolVM({ image: 'python:3.12-alpine', net: false });
  const result = await vm.run(`python3 -c "${code.replace(/"/g, '\\"')}"`);
  await vm.destroy();
  return result;
}

const result = await runUntrustedCode('print("Hello from isolated VM!")');
console.log(result.stdout);  // Hello from isolated VM!
```

### Persistent dev VM across tool calls

```js
const vm = new SmolVM({ image: 'node:22-slim', net: true });
await vm.start();

// Multiple commands — state is preserved between exec() calls
await vm.exec('npm install -g typescript');
await vm.exec('echo "const x: number = 1" > /tmp/test.ts');
const result = await vm.exec('npx tsc --noEmit /tmp/test.ts');

await vm.stop();
```

### Egress-controlled VM (safe API access)

```js
const vm = new SmolVM({
  image: 'alpine',
  net: true,
  allowHosts: ['api.openai.com'],
  env: { OPENAI_API_KEY: process.env.OPENAI_API_KEY },
});
const result = await vm.run('wget -qO- https://api.openai.com/v1/models');
// google.com would be blocked, only api.openai.com is reachable
await vm.destroy();
```

### SSH agent forwarding for private git access

```js
const vm = new SmolVM({
  image: 'alpine',
  net: true,
  sshAgent: true,  // SSH_AUTH_SOCK must be set on host
});
await vm.start();
await vm.exec('apk add -q openssh-client git');
await vm.exec('git clone git@github.com:org/private-repo.git /tmp/repo');
await vm.stop();
```

---

## Notes & Limitations

- **File writes**: write to mounted volumes (`/workspace` etc.), not to the container rootfs.
  See the main SKILL.md Known Limitations section.
- **Network**: `net: false` by default. ICMP (ping) not supported even with `net: true`.
- **Volume mounts**: directories only, not single files.
- **SSH agent**: requires `SSH_AUTH_SOCK` to be set on the host process.
- For the definitive and most current API, read `lib/js/` in the GitHub repo.

---

## Links

- Source: https://github.com/smol-machines/smolvm/tree/main/lib/js
- Issues: https://github.com/smol-machines/smolvm/issues
- Discord: https://discord.gg/vx375Jyn
