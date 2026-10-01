# 0001 — Use OpenShell's VM driver (libkrun), not Docker

**Date:** 2026-10-01 · **Status:** accepted

## Context

OpenShell runs sandboxes through a compute driver. On macOS, the installer's default is Docker.

- On a Mac, Docker Desktop runs every container inside one shared Linux VM. All sandboxes share
  that VM's kernel, plus whatever is mounted into it (often `/Users`).
- OpenShell's Docker driver starts sandbox containers with `network=host` and points the
  supervisor at `https://127.0.0.1:17670`. On Docker Desktop, "host" is Docker's VM, not the Mac,
  so the supervisor never reaches the gateway (`Startup configuration fetch failed`).

## Decision

Use the **VM driver**:

- one libkrun microVM per sandbox, with its own kernel
- no network card in the VM, only vsock to the supervisor
- the supervisor runs as a native Mac process

The config is versioned in [`openshell/gateway.toml`](../../openshell/gateway.toml).

## Consequences

- A kernel exploit inside a sandbox stays inside that sandbox's VM.
- Docker is not needed to run sandboxes. A container engine is only needed to *build* custom
  images locally.
- The VM driver is marked experimental in OpenShell 0.1.2. Expect rough edges, such as the README
  being wrong about the `guest_tls_*` keys.
- It requires `e2fsprogs` from Homebrew.
