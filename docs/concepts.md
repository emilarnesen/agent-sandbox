# Concepts

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

## Why libkrun and not Docker

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

The full reasoning is in [decisions/0001-vm-driver.md](decisions/0001-vm-driver.md).

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
