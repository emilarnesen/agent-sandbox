# agent-sandbox

A hands-on lab for running AI agents in isolated sandboxes on a Mac mini (Apple silicon), using
[NVIDIA OpenShell](https://github.com/NVIDIA/openshell).

The goal is to learn how to give an agent **exactly** the access it needs and nothing more:

- which files it can see
- which hosts it can reach
- which credentials it can use

All of this is enforced by something the agent cannot tamper with. Each sandbox is its own
**libkrun microVM**, with its own Linux kernel and no network card. The agent never sees real API
keys. The lessons are meant to carry over to agent platforms at work (Linux, OpenShift, Kata
Containers).

## Quickstart

```sh
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
brew install e2fsprogs
cp openshell/gateway.toml /opt/homebrew/var/openshell/gateway.toml && brew services restart openshell
openshell profile import -f openshell/profiles/openrouter.yaml
```

Then create a provider and a sandbox, as described in [docs/usage.md](docs/usage.md).

## Documentation

| Doc | Contents |
|---|---|
| [docs/concepts.md](docs/concepts.md) | How it works, why libkrun instead of Docker, glossary |
| [docs/setup-macos.md](docs/setup-macos.md) | Installation, gateway config explained, troubleshooting |
| [docs/usage.md](docs/usage.md) | Profiles, providers, sandboxes, rules and policy |
| [docs/decisions/](docs/decisions/) | Architecture decisions (ADRs) |

## Repository layout

```
docs/                  documentation and decisions
openshell/
  gateway.toml         gateway config for macOS + VM driver
  profiles/            provider profiles (what a key may reach, and by which program)
```

Planned: `images/` for custom sandbox images, and `.github/workflows/` for testing sandboxed agents
on runners.

## Links

- OpenShell docs: https://docs.nvidia.com/openshell/latest/
- OpenShell repo: https://github.com/NVIDIA/openshell
- VM driver README: https://github.com/NVIDIA/OpenShell/blob/main/crates/openshell-driver-vm/README.md
- Policies: https://docs.nvidia.com/openshell/latest/how-it-works/policies/overview
- Kaiden: https://openkaiden.ai/
