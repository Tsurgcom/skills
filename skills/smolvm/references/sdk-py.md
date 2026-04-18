# smolvm Python SDK Reference

> **Source**: `lib/py/` in the smol-machines/smolvm GitHub repository.
> This reference is derived from the repo README and examples. The `lib/py/` directory
> is the canonical source — always check there for the most current API.

## Overview

The Python SDK lets you spawn and control microVMs programmatically from Python.
It wraps the same underlying libkrun/crun stack as the CLI.

Use cases:
- Embed a sandboxed Python execution environment in an app or agent framework
- Safely run LLM-generated code without risking the host
- Build test harnesses that spin up fresh VMs per test case
- Integrate with agent frameworks (LangChain, smolagents, etc.)

## Installation

The Python library lives in `lib/py/` in the repo. As of April 2026 it is intended
to be used directly or vendored — check the repo and PyPI for any published package.

```bash
# From the repo root
cd lib/py
pip install -e .   # installs in editable mode

# Or copy lib/py/ into your project and import directly
```

> Note: `smolvm-core` on PyPI is a **different project** (CelestoAI/SmolVM) and is
> NOT the official smol-machines Python SDK.

## Quick Start

```python
from smolvm import SmolVM

# Ephemeral VM — runs one command, then cleans up
vm = SmolVM(image="alpine:latest", net=True)
result = vm.run("echo hello from a microVM")
print(result.stdout)   # hello from a microVM
vm.destroy()

# Context manager (auto-destroys on exit)
with SmolVM(image="python:3.12-alpine") as vm:
    result = vm.run('python3 -c "import sys; print(sys.version)"')
    print(result.stdout)
```

## API Reference

---

### `SmolVM(options)` constructor

```python
vm = SmolVM(
    image="python:3.12-alpine",   # OCI container image (required)
    net=False,                    # enable outbound TCP/UDP networking
    allow_hosts=["api.stripe.com"],  # egress allowlist (requires net=True)
    volumes=["/tmp:/workspace"],  # HOST:GUEST directory mounts
    env={"MY_KEY": "my_value"},  # environment variables
    ssh_agent=False,             # forward host SSH agent
    cpus=4,                      # vCPU count (default: 4)
    mem=8192,                    # RAM in MiB (default: 8192)
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `image` | str | required | OCI container image |
| `net` | bool | `False` | Enable outbound networking |
| `allow_hosts` | list[str] | `[]` | Allowlist for egress (requires `net=True`) |
| `volumes` | list[str] | `[]` | `HOST:GUEST` directory mounts (directories only) |
| `env` | dict[str, str] | `{}` | Environment variables injected into the guest |
| `ssh_agent` | bool | `False` | Forward host SSH agent (requires `SSH_AUTH_SOCK`) |
| `cpus` | int | `4` | Number of vCPUs |
| `mem` | int | `8192` | RAM in MiB |

---

### `vm.run(command) -> RunResult`

Run a shell command in an **ephemeral** VM (created, runs, destroyed). Returns a `RunResult`.

```python
result = vm.run("python3 --version")
print(result.stdout)    # Python 3.12.x
print(result.stderr)    # (empty)
print(result.exit_code) # 0
```

**`RunResult` attributes:**

```python
@dataclass
class RunResult:
    stdout: str
    stderr: str
    exit_code: int
```

---

### `vm.start() -> None`

Start a **persistent** named VM (continues running until `vm.stop()`).

```python
vm.start()
```

---

### `vm.exec(command) -> RunResult`

Run a command inside an **already-running** persistent VM. Returns `RunResult`.

```python
vm.start()
vm.exec("apk add git")
result = vm.exec("git --version")
print(result.stdout)
```

---

### `vm.stop() -> None`

Stop a running persistent VM.

```python
vm.stop()
```

---

### `vm.destroy() -> None`

Stop and fully clean up the VM. For ephemeral VMs called after `run()`; for persistent
VMs, equivalent to stop + remove.

```python
vm.destroy()
```

---

### Context manager protocol

`SmolVM` supports Python's `with` statement. The VM is destroyed automatically on exit.

```python
with SmolVM(image="alpine", net=True) as vm:
    result = vm.run("uname -a")
    print(result.stdout)
# VM destroyed here automatically
```

---

### `SmolVM.pack(image, output, single_file=False)` — build portable executables

```python
SmolVM.pack(
    image="python:3.12-alpine",
    output="./python312",
    single_file=False,   # if True: single binary, no .smolmachine sidecar
)
# Produces: ./python312 + ./python312.smolmachine
```

---

## Usage Patterns

### Sandboxed code execution for AI agents

```python
from smolvm import SmolVM

def run_agent_code(code: str) -> str:
    """Run LLM-generated code in a hardware-isolated VM."""
    with SmolVM(image="python:3.12-alpine", net=False) as vm:
        result = vm.run(f'python3 -c "{code}"')
        if result.exit_code != 0:
            return f"Error: {result.stderr}"
        return result.stdout
```

### Persistent VM for multi-step agent workflows

```python
vm = SmolVM(image="python:3.12-alpine", net=True)
vm.start()

# State persists between exec() calls
vm.exec("pip install requests pandas")
vm.exec("python3 -c 'import requests; print(\"requests installed\")'")
result = vm.exec("python3 my_analysis.py")

vm.stop()
```

### Egress-controlled VM (safe external API access)

```python
import os

with SmolVM(
    image="python:3.12-alpine",
    net=True,
    allow_hosts=["api.openai.com"],
    env={"OPENAI_API_KEY": os.environ["OPENAI_API_KEY"]},
) as vm:
    # Only api.openai.com is reachable; all other hosts are blocked
    result = vm.run("python3 -c \"import urllib.request; print(urllib.request.urlopen('https://api.openai.com/v1/models').status)\"")
    print(result.stdout)
```

### Filesystem isolation — write to mounted volumes

```python
import tempfile, os

with tempfile.TemporaryDirectory() as tmp:
    with SmolVM(
        image="python:3.12-alpine",
        volumes=[f"{tmp}:/workspace"],
    ) as vm:
        vm.run("python3 -c \"open('/workspace/output.txt', 'w').write('result')\"")
    
    # Read result back on the host
    print(open(os.path.join(tmp, "output.txt")).read())  # "result"
```

### SSH agent forwarding for private git access

```python
with SmolVM(image="alpine", net=True, ssh_agent=True) as vm:
    vm.run("apk add -q openssh-client git")
    result = vm.run("ssh-add -l")  # lists host keys but they can't be extracted
    vm.run("git clone git@github.com:org/private-repo.git /tmp/repo")
```

### Integration with agent frameworks

```python
# As a tool for LangChain / smolagents / any framework
def sandbox_tool(code: str) -> str:
    """Execute Python code safely in a microVM sandbox."""
    with SmolVM(image="python:3.12-alpine", net=False) as vm:
        result = vm.run(f'python3 << "EOF"\n{code}\nEOF')
        return result.stdout if result.exit_code == 0 else result.stderr
```

---

## Notes & Limitations

- **File writes**: write to mounted volumes (`/workspace` etc.), not to the container rootfs.
  Writes to `/tmp`, `/home` etc. inside the VM rootfs will fail due to a libkrun TSI bug.
- **Network**: `net=False` by default. ICMP (`ping`) not supported even with `net=True`.
- **Volume mounts**: directories only, not individual files. Use the **top-level** mount path —
  nested paths (e.g. `/mnt/data`) can trigger the rootfs write bug.
- **SSH agent**: requires `SSH_AUTH_SOCK` to be set on the host process.
- **Overlay space**: persistent VMs have a ~2 GiB writable overlay. Heavy installs can fill it.
- For the definitive and most current API, read `lib/py/` in the GitHub repo.

---

## Links

- Source: https://github.com/smol-machines/smolvm/tree/main/lib/py
- Issues: https://github.com/smol-machines/smolvm/issues
- Discord: https://discord.gg/vx375Jyn
