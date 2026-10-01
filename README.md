# agent-sandbox

A hands-on lab for running AI agents in isolated sandboxes on a Mac mini (Apple silicon), using
[NVIDIA OpenShell](https://github.com/NVIDIA/openshell).

The goal is to learn how to give an agent **exactly** the access it needs and nothing more:

- which files it can see
- which hosts it can reach
- which credentials it can use

All of this is enforced by something the agent cannot tamper with. The lessons are meant to carry
over to agent platforms at work (Linux, OpenShift, Kata Containers).

## How it works

```
macOS (host)
 ├─ OpenShell gateway      control plane: stores profiles, providers (keys) and policy
 ├─ OpenShell supervisor   native Mac process: DNS, policy checks, all outbound traffic, key injection
 └─ microVM per sandbox    libkrun, own Linux kernel, NO network card
     └─ agent (OpenCode)   reaches the outside world only by asking the supervisor over vsock
```

The agent never holds a real API key. It sees a placeholder, and the supervisor swaps in the real
key on the way out, only for the endpoint the key belongs to. Any connection the policy does not
allow is blocked and shows up as a **pending rule** that you can inspect and approve or reject.

### Why libkrun and not Docker

OpenShell can run sandboxes as Docker containers or as microVMs. We use the **VM driver
(libkrun)**.

- **What libkrun is:** a small library that turns a process into a lightweight VM. It uses Apple's
  Hypervisor.framework on macOS, or KVM on Linux. Each sandbox gets its own Linux kernel and boots
  in about a second.
- **Why not Docker:** on a Mac, every Docker container runs inside one shared Docker Desktop
  Linux VM, with one shared kernel. A kernel exploit inside one sandbox reaches that VM, all other
  containers, and whatever is mounted into it. With libkrun, a kernel exploit only breaks that
  sandbox's own tiny VM.
- **A bonus:** the libkrun VM has no network card at all, only a vsock channel to the
  supervisor. The supervisor runs on the Mac, outside the VM.
- **A practical reason:** OpenShell's Docker driver gives containers `network=host` and
  `127.0.0.1`. That breaks on Docker Desktop, because "host" there is Docker's VM, not the Mac.
  Docker is not needed at all with the VM driver.

## Glossary

| Term | What it is |
|---|---|
| **OpenShell** | NVIDIA's open-source runtime for running agents in sandboxes that a policy controls. The CLI is `openshell`. |
| **Gateway** | OpenShell's control plane, running as a Homebrew service on `https://127.0.0.1:17670`. It is an API, not a web page, and it requires mTLS. |
| **Supervisor** | The trusted "judge" between the sandbox and the world. It checks every request against policy and injects credentials. |
| **Sandbox** | One isolated environment holding one agent. In our setup, that means one microVM. |
| **Compute driver** | How OpenShell runs sandboxes: `docker`, `podman`, `kubernetes` or `vm`. We use `vm`. |
| **libkrun** | A library that turns a process into a lightweight VM, using Hypervisor.framework or KVM. It is what the `vm` driver is built on. |
| **microVM** | A minimal VM: its own kernel, few devices, fast boot. It is a stronger boundary than a container. |
| **vsock** | A private VM ↔ host socket with no networking involved. It is the sandbox's only way out. |
| **Hypervisor.framework** | Apple's low-level API for running VMs on macOS. libkrun uses it. |
| **e2fsprogs** | Linux filesystem tools (`mkfs.ext4`). The VM driver needs them to build the VM's disk on macOS. |
| **Bootstrap image** | The base image the microVM boots from (`nvcr.io/nvidia/base/ubuntu:24.04`). The agent image is unpacked on top of it. |
| **OCI image** | The standard container image format, not tied to Docker. Both drivers use the same images. |
| **Profile** | A template describing a service: which key it needs, which hosts the key may go to, and which programs may use it. It contains no secrets. |
| **Provider** | A profile plus your actual key, stored in the gateway. You attach it to a sandbox with `--provider`. |
| **Inference** | Running a language model to get answers. "Inference provider" means a model API. |
| **Credential placeholder** | What the agent sees instead of the key, for example `openshell:resolve:env:…_OPENROUTER_API_KEY`. |
| **Policy** | The sandbox's rulebook: readable and writable paths, and which program may reach which host. |
| **Rule / chunk** | A proposed policy change, created when the sandbox tries something the policy does not allow. You approve or reject it. |
| **Landlock / seccomp** | Linux kernel features inside the sandbox. Landlock restricts files; seccomp restricts system calls. |
| **mTLS** | Mutual TLS: both client and server present certificates. It is how the CLI and supervisor prove who they are to the gateway. |
| **OpenCode** | An open-source terminal coding agent, like Claude Code, that can use any model vendor. It is our test agent. |
| **OpenRouter** | One API and one key giving access to many models, some of them free. It is the agent's "brain" for now. |
| **Kaiden** | A desktop app from Red Hat for managing agents. It uses OpenShell underneath. Not used yet. |
| **agent-sandbox (k8s)** | `kubernetes-sigs/agent-sandbox`: Kubernetes CRDs for agent sandboxes. It is the Linux/OpenShift track, unrelated to this repo's name. |
| **Kata Containers** | VM-per-pod for Kubernetes, the same idea as libkrun. It is what OpenShift sandboxed containers use. |

## Getting started

Requirements: a Mac with Apple silicon, Homebrew, and an [OpenRouter](https://openrouter.ai/keys)
API key. Docker is **not** required.

**1. Install OpenShell.** The script installs a Homebrew formula from the `nvidia/openshell` tap,
which includes the CLI, the gateway and the VM driver. It also starts the gateway as a
`brew services` service.

```sh
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
brew install e2fsprogs      # needed by the VM driver to build ext4 disks
```

**2. Switch the gateway to the VM driver.** Edit `/opt/homebrew/var/openshell/gateway.toml`:

```toml
[openshell]
version = 2

[openshell.gateway]
compute_driver = "vm"
guest_tls_ca   = "/opt/homebrew/var/openshell/tls/ca.crt"
guest_tls_cert = "/opt/homebrew/var/openshell/tls/client/tls.crt"
guest_tls_key  = "/opt/homebrew/var/openshell/tls/client/tls.key"

[openshell.drivers.vm]
grpc_endpoint = "https://127.0.0.1:17670"
driver_dir    = "/opt/homebrew/opt/openshell/libexec"
state_dir     = "/opt/homebrew/var/openshell/vm-driver"
```

> **Gotcha (v0.1.2):** the driver README puts `guest_tls_ca` under `[openshell.drivers.vm]`. The
> gateway rejects that. All three `guest_tls_*` keys must be under `[openshell.gateway]`, and all
> three are required.

**3. Restart and verify:**

```sh
brew services restart openshell
openshell status            # Status: Connected
openshell gateway info      # Compute drivers: vm
```

If the gateway does not come up, the logs are at
`/opt/homebrew/var/log/openshell/openshell-gateway.err.log`.

## Working with OpenShell

### Profiles: describe a service

```sh
openshell profile import --url https://raw.githubusercontent.com/NVIDIA/OpenShell/main/providers/openrouter.yaml
openshell profile list
openshell profile describe openrouter     # readable summary
openshell profile export openrouter       # full YAML
```

The OpenRouter profile says three things:

- the key is `OPENROUTER_API_KEY`, sent as a bearer token
- it may only go to `openrouter.ai:443`
- only `/usr/local/bin/opencode` may use it

The `binaries` list must match the program paths in *your* image. Copy and edit the profile
rather than importing it unchanged when you use other images.

### Providers: store a key

```sh
read -s "OPENROUTER_API_KEY?OpenRouter key: "; export OPENROUTER_API_KEY   # keeps it out of shell history
openshell provider create --name openrouter --type openrouter --from-existing
unset OPENROUTER_API_KEY                                                  # now only the gateway has it
openshell provider list
```

`--from-existing` reads the key from your current shell environment.

### Sandboxes: run an agent

```sh
openshell sandbox create --name agent-0001 \
  --from ghcr.io/anomalyco/opencode:latest \
  --provider openrouter \
  -- opencode -m openrouter/nvidia/nemotron-3.5-lightning:free

openshell sandbox list
openshell sandbox connect agent-0001     # re-attach
openshell sandbox exec agent-0001 -- uname -a
openshell sandbox stop|start|delete agent-0001
openshell logs agent-0001
```

The first create is slow, because the driver downloads the images and builds VM disks under
`state_dir`. Free OpenRouter models are slow too (queues and rate limits), but the sandbox itself
adds only milliseconds.

**What the walls look like from inside** (tested on agent-0001):

| Inside the sandbox | Result | Meaning |
|---|---|---|
| `uname -a` | `Linux agent-0001 6.12.76 … aarch64` | Its own kernel. |
| `echo $OPENROUTER_API_KEY` | `openshell:resolve:env:…` | A placeholder, never the real key. |
| `ls /Users` | No such file or directory | No access to the Mac's files. |
| `ip addr` | `lo` only | No network card. |
| `wget https://example.com` | `198.18.0.6 … Permission denied` | DNS answered by the supervisor with a fake address, and the connection is blocked by policy. |

### Rules and policy: audit and approve

Every blocked connection becomes a proposed rule (a "chunk"). The rule records:

- which program tried
- which host and port it tried to reach
- a rationale and confidence score
- a prover check

```sh
openshell rule get agent-0001 --status pending
openshell rule approve agent-0001 --chunk-id <id>     # takes effect immediately, no restart
openshell rule reject  agent-0001 --chunk-id <id>
openshell rule history agent-0001                     # audit trail
openshell policy get agent-0001 --full                # the effective policy as YAML
```

Approved rules are narrow. Approving `wget` → `example.com` produced a rule that allows only
`/bin/busybox` to reach `example.com:443`, nothing broader.

Auditing also reveals what an agent does on its own. At startup, OpenCode tried to reach
`api.github.com`, `registry.npmjs.org` and `models.opencode.ai`, most likely for update checks and
model lists. All three were blocked and are waiting as pending rules.

## Links

- OpenShell docs: https://docs.nvidia.com/openshell/latest/
- OpenShell repo: https://github.com/NVIDIA/openshell
- VM driver README: https://github.com/NVIDIA/OpenShell/blob/main/crates/openshell-driver-vm/README.md
- Policies: https://docs.nvidia.com/openshell/latest/how-it-works/policies/overview
- Kaiden: https://openkaiden.ai/
