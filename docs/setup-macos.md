# Setup on macOS

Requirements: a Mac with Apple silicon, Homebrew, and an [OpenRouter](https://openrouter.ai/keys)
API key. Docker is **not** required.

## 1. Install OpenShell

The script installs a Homebrew formula from the `nvidia/openshell` tap, which includes the CLI, the
gateway and the VM driver. It also starts the gateway as a `brew services` service.

```sh
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
brew install e2fsprogs      # needed by the VM driver to build ext4 disks
```

## 2. Switch the gateway to the VM driver

Copy the versioned config from this repo into place:

```sh
cp openshell/gateway.toml /opt/homebrew/var/openshell/gateway.toml
```

It contains:

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

| Key | Why |
|---|---|
| `compute_driver = "vm"` | Use libkrun microVMs instead of Docker. |
| `guest_tls_*` | The CA, certificate and key the supervisor uses to authenticate to the gateway over mTLS. These are the client certificates Homebrew generated. |
| `grpc_endpoint` | Required. How the supervisor (a native Mac process) reaches the gateway. |
| `driver_dir` | Where Homebrew puts the VM driver. The gateway does not search there by default. |
| `state_dir` | VM disks, image cache and console logs. |

> **Gotcha (v0.1.2):** the driver README puts `guest_tls_ca` under `[openshell.drivers.vm]`. The
> gateway rejects that. All three `guest_tls_*` keys must be under `[openshell.gateway]`, and all
> three are required.

## 3. Restart and verify

```sh
brew services restart openshell
openshell status            # Status: Connected
openshell gateway info      # Compute drivers: vm
```

## Troubleshooting

| Symptom | Cause |
|---|---|
| `openshell status` → `Connection refused` | The gateway is crash-looping on a bad config. Check `/opt/homebrew/var/log/openshell/openshell-gateway.err.log`. |
| `ProvisioningFailed … mke2fs not found` | Run `brew install e2fsprogs`. |
| Docker driver: `Startup configuration fetch failed` | The container uses `network=host` and `127.0.0.1`, which on Docker Desktop is Docker's VM, not the Mac. Use the VM driver. |
| Browser can't open `localhost:17670` | Expected. It is an mTLS API, not a web page. |
