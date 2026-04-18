---
name: smolvm
description: >
  Expert guidance for smolvm — the open-source CLI tool (YC S26) for running microVMs locally
  on macOS and Linux. Use this skill whenever the user mentions smolvm, microVMs, smolmachines,
  sandboxing workloads, running containers in isolated VMs, packaging portable self-contained
  executables (.smolmachine), or asks about lightweight VM alternatives to Docker/QEMU/Firecracker/Kata.
  Also trigger for: safely running coding agents locally, VM boot performance, egress network
  filtering, SSH agent forwarding into VMs, embedding VMs in a single binary, Smolfile config,
  or the smolvm JS/Python SDK. Use even if the user just says "smolvm", "microvm", or
  "smolmachine" in any context.
---

# smolVM Skill

smolVM is an open-source CLI tool (written in Rust) that makes microVMs easy to run locally.
It wraps [libkrun](https://github.com/containers/libkrun) + a custom kernel (libkrunfw) +
Hypervisor.framework (macOS) / KVM (Linux) + the crun container runtime.

- **Boot time**: <200ms
- **Defaults**: 4 vCPUs, 8 GiB RAM (elastic via virtio balloon — host commits only what the guest uses)
- **Distribution**: single binary, no daemon
- **Status**: Alpha (v0.1.x); APIs can change. [Report issues](https://github.com/smol-machines/smolvm/issues)
- **Community**: [Discord](https://discord.gg/vx375Jyn)

> For SDK libraries (JS / Python) see `lib/js/` and `lib/py/` in the GitHub repo, and the
> reference files `references/sdk-js.md` and `references/sdk-py.md` in this skill folder.

---

## Install & Uninstall

```bash
# Install (macOS + Linux)
curl -sSL https://smolmachines.com/install.sh | bash

# Recommended: install and then discover all commands
curl -sSL https://smolmachines.com/install.sh | bash && smolvm --help

# Uninstall
curl -sSL https://smolmachines.com/install.sh | bash -s -- --uninstall
```

Alternatively, download a pre-built binary directly from
[GitHub Releases](https://github.com/smol-machines/smolvm/releases).

> **macOS**: the binary must be code-signed with Hypervisor.framework entitlements.
> The install script handles this automatically.

---

## Platform Support

| Host | Guest | Requirements |
|------|-------|-------------|
| macOS Apple Silicon | arm64 Linux | macOS 11+ |
| macOS Intel | x86_64 Linux | macOS 11+ (untested) |
| Linux x86_64 | x86_64 Linux | KVM (`/dev/kvm`) |
| Linux aarch64 | aarch64 Linux | KVM (`/dev/kvm`) |

---

## Command Reference

smolvm has three top-level commands: `machine`, `pack`, and legacy aliases.

### `smolvm machine` — unified VM management

#### Ephemeral VM (cleaned up on exit)

```bash
# Run a command in a fresh VM — destroyed when done
smolvm machine run --net --image alpine -- sh -c "echo 'Hello from a microVM' && uname -a"

# Interactive shell
smolvm machine run --net -it --image alpine -- /bin/sh
# Inside: apk add sl && sl && exit

# With a volume mount
smolvm machine run --net -v /tmp:/workspace --image alpine -- ls /workspace

# Restrict egress to specific hosts only
smolvm machine run --net --image alpine \
  --allow-host registry.npmjs.org -- \
  wget -q -O /dev/null https://registry.npmjs.org   # allowed

smolvm machine run --net --image alpine \
  --allow-host registry.npmjs.org -- \
  wget -q -O /dev/null https://google.com            # blocked — not in allow list

# SSH agent forwarding (private keys never enter the VM)
smolvm machine run --ssh-agent --net --image alpine -- \
  sh -c "apk add -q openssh-client && ssh-add -l"

# Override CPU and memory
smolvm machine run --net --image alpine --cpus 2 --mem 2048 -- uname -a
```

**Key flags for `machine run`:**

| Flag | Meaning |
|------|---------|
| `--image <img>` | Container image to use (e.g. `alpine`, `python:3.12-alpine`) |
| `--net` | Enable outbound networking (TCP/UDP only; ICMP/ping not supported) |
| `--allow-host <host>` | Restrict egress to this host only (repeatable); requires `--net` |
| `-v HOST:GUEST` | Mount a host directory (directories only, not single files) |
| `-it` | Interactive TTY session |
| `-e KEY=VALUE` | Inject environment variable |
| `--ssh-agent` | Forward host SSH agent; requires `SSH_AUTH_SOCK` set on host |
| `--cpus <n>` | Override vCPU count (default: 4) |
| `--mem <mb>` | Override RAM in MiB (default: 8192) |

#### Persistent named VM

State (installed packages, files in overlay) survives across restarts.

```bash
# Create
smolvm machine create --net myvm

# Start
smolvm machine start --name myvm

# Run commands inside
smolvm machine exec --name myvm -- apk add git
smolvm machine exec --name myvm -it -- /bin/sh

# Clone a private repo via forwarded SSH agent
smolvm machine exec --name myvm -- git clone git@github.com:org/private-repo.git

# Stop
smolvm machine stop --name myvm
```

> Installed packages go to the overlay layer (sparse disk, /dev/vdb on the guest).
> The overlay starts at 2 GiB — installing heavy tools like Claude Code + Codex together
> can exhaust this. This is a known limitation being addressed.

---

### `smolvm pack` — portable executable VM images

Bundles a container image into a self-contained executable. The result runs anywhere the
host architecture matches, with zero dependencies.

```bash
# Two-file output: ./my-sandbox (launcher) + ./my-sandbox.smolmachine (rootfs)
smolvm pack create --image alpine -o ./my-sandbox

# Single-file output (larger binary, no sidecar .smolmachine)
smolvm pack create --image alpine -o ./my-sandbox --single-file

# Run the packed VM
./my-sandbox uname -a            # runs inside the guest Linux VM
./my-sandbox echo "hello"

# Pack a specific runtime
smolvm pack create --image python:3.12-alpine -o ./python312
./python312 run -- python3 --version   # Python 3.12.x, isolated, no pyenv/venv needed

# Pack a Node.js environment
smolvm pack create --image node:22-slim -o ./nodebox
./nodebox run -- node --version
```

> `./my-sandbox.smolmachine` is a stateful VM image — pack it with the launcher to
> rehydrate it on any supported platform.

---

## Smolfile — Reproducible VM Config

A `Smolfile` is a TOML file that declares a machine's full environment. Use it to version-control
your VM setup alongside your code.

```toml
# Smolfile
image = "python:3.12-alpine"
net = true

[network]
allow_hosts = ["api.stripe.com", "db.example.com"]

[dev]
init = ["pip install -r requirements.txt"]
volumes = ["./src:/app"]

[auth]
ssh_agent = true
```

```bash
# Create and start a machine from a Smolfile
smolvm machine create myvm -s Smolfile
smolvm machine start --name myvm
```

**Smolfile fields:**

| Field | Type | Description |
|-------|------|-------------|
| `image` | string | Container image |
| `net` | bool | Enable networking |
| `[network].allow_hosts` | list of strings | Egress allowlist (requires `net = true`) |
| `[dev].init` | list of commands | Run once on first start (install deps, etc.) |
| `[dev].volumes` | list of `HOST:GUEST` | Directory mounts |
| `[auth].ssh_agent` | bool | Forward host SSH agent |

---

## Common Use Cases

### Safely running coding agents

```bash
# Agent is fully isolated; host filesystem, network and creds are behind hypervisor boundary
smolvm machine run --net -v /tmp/agent-workspace:/workspace \
  --image python:3.12-alpine -- python3 /workspace/agent.py

# Run Claude Code inside a persistent VM
smolvm machine run --net \
  -e "ANTHROPIC_API_KEY=sk-ant-..." \
  --image node:22-slim -- \
  npx -y @anthropic-ai/claude-code -p "Just say Hello, World! and exit."
```

### Distributing sandboxed CLIs

```bash
smolvm pack create --image python:3.12-alpine -o ./my-tool --single-file
# Ship ./my-tool — recipients need no Python, Docker, or any runtime installed
```

### Locked-down egress (no exfiltration)

```bash
# Network is off by default — untrusted code cannot phone home
smolvm machine run --image alpine -- ping -c 1 1.1.1.1    # fails

# Allow only what you need
smolvm machine run --net --image alpine \
  --allow-host api.openai.com -- curl https://api.openai.com/v1/models
```

### SSH agent forwarding for git operations

```bash
smolvm machine run --ssh-agent --net --image alpine -- \
  sh -c "apk add -q openssh-client && ssh-add -l"
# Keys are listed but cannot be extracted from the VM
```

### Interactive dev environment

```bash
smolvm machine create --net devbox
smolvm machine start --name devbox
smolvm machine exec --name devbox -- apk add git vim python3
smolvm machine exec --name devbox -it -- /bin/sh   # persistent shell
smolvm machine stop --name devbox
```

---

## Architecture

```
smolvm binary
└── libkrun (VMM library, no separate daemon)
    ├── libkrunfw (custom Linux kernel)
    ├── Hypervisor.framework (macOS) / KVM (Linux)
    └── crun (container runtime inside the guest)
```

- **No daemon**: the VMM is a library linked into the smolvm binary itself.
- **Memory**: elastic via virtio balloon. Host only commits pages the guest actually touches.
  vCPU threads sleep in the hypervisor when idle — over-provisioning is near-free.
- **Storage**: two sparse virtual disks per VM:
  - `/dev/vda` (storage disk): OCI image layers + container overlays
  - `/dev/vdb` (overlay disk): writable diffs (apk/pip installs etc.), default 2 GiB

---

## Comparison

| | Containers | Colima | QEMU | Firecracker | Kata | smolvm |
|---|---|---|---|---|---|---|
| Kernel isolation | shared | 1 VM | separate | separate | per container | VM per workload |
| Boot time | ~100ms | ~seconds | ~15-30s | <125ms | ~500ms | **<200ms** |
| Setup | easy | easy | complex | complex | complex | **easy** |
| macOS native | via Docker VM | yes (krunkit) | yes | ❌ | ❌ | ✅ |
| Embeddable SDK | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Single binary | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## Known Limitations & Workarounds

### File writes inside the container rootfs fail

**Cause**: libkrun TSI bug with overlayfs. Writes to `/tmp`, `/home`, etc. inside the container
rootfs will fail.

**Workaround**: always write to a mounted volume, using the **top-level** mount path.

```bash
# ✅ Works — top-level mount path
smolvm machine run --net -v /tmp:/workspace --image alpine -- \
  sh -c "echo 'hello' > /workspace/out.txt"

# ❌ Fails — nested mount path (/mnt/data, not /data)
smolvm machine run --net -v /tmp:/mnt/data --image alpine -- \
  sh -c "echo 'hello' > /mnt/data/out.txt"

# ❌ Fails — writing to container rootfs
smolvm machine run --image alpine -- sh -c "echo 'hi' > /tmp/test.txt"
```

### Networking: TCP/UDP only

ICMP (`ping`) and raw sockets do not work even with `--net`.

### Volume mounts: directories only

Single-file mounts are not supported. Mount a directory and reference files within it.

### SSH agent forwarding prerequisite

`--ssh-agent` requires an SSH agent running on the host (`SSH_AUTH_SOCK` must be set).
Check with `ssh-add -l`.

### Overlay disk space

The writable overlay layer is 2 GiB by default. Installing large tools (e.g. both Codex and
Claude Code) can fill it. This is a known bug being fixed. In the meantime, prefer ephemeral
`machine run` sessions with pre-built images that already include your deps.

### macOS code signing

The binary must be signed with Hypervisor.framework entitlements. The install script handles
this. If building from source, the `.entitlements` file is in the repo root.

---

## Development & Troubleshooting

```bash
# Build from source
./scripts/build-dist.sh

# Install locally after build
./scripts/install-local.sh

# Run tests
./tests/run_all.sh
```

**"Database already open" / lock errors:**
```bash
pkill -f "smolvm serve"
pkill -f "smolvm-bin microvm start"
```

**Hung tests — check for stuck VM processes:**
```bash
ps aux | grep smolvm
```

**Volume not visible inside guest** — this was a bug fixed in v0.1.18+. If mounts show
"Mounts: 1" in the output but the directory isn't visible, upgrade to the latest release or
build from main.

---

## SDK Libraries

smolvm ships JS (Node.js) and Python libraries in the `lib/` directory of the GitHub repo,
enabling you to embed and drive microVMs programmatically from your own apps.

→ See `references/sdk-js.md` for the JavaScript/Node.js API reference.
→ See `references/sdk-py.md` for the Python API reference.

The comparison table lists **"Embeddable SDK: yes"** as a unique smolvm feature vs. all
alternatives — these libraries are the mechanism for that.

---

## Useful Links

- GitHub: https://github.com/smol-machines/smolvm
- Website / install script: https://smolmachines.com
- Discord: https://discord.gg/vx375Jyn
- Report issues: https://github.com/smol-machines/smolvm/issues
- Releases: https://github.com/smol-machines/smolvm/releases
- YC listing: https://www.ycombinator.com/companies/smol-machines
- License: Apache-2.0
