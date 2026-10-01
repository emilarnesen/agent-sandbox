# Using OpenShell

## Profiles: describe a service

```sh
openshell profile import -f openshell/profiles/openrouter.yaml   # or --url <upstream yaml>
openshell profile list
openshell profile describe openrouter     # readable summary
openshell profile export openrouter       # full YAML
openshell profile update -f openshell/profiles/openrouter.yaml   # after editing
```

The OpenRouter profile ([openshell/profiles/openrouter.yaml](../openshell/profiles/openrouter.yaml))
says three things:

- the key is `OPENROUTER_API_KEY`, sent as a bearer token
- it may only go to `openrouter.ai:443`
- only `/usr/local/bin/opencode` may use it

The `binaries` list must match the program paths in *your* image. Copy and edit the profile when
you use other images.

## Providers: store a key

```sh
read -s "OPENROUTER_API_KEY?OpenRouter key: "; export OPENROUTER_API_KEY   # keeps it out of shell history
openshell provider create --name openrouter --type openrouter --from-existing
unset OPENROUTER_API_KEY                                                  # now only the gateway has it
openshell provider list
```

`--from-existing` reads the key from your current shell environment.

## Sandboxes: run an agent

```sh
openshell sandbox create --name agent-0001 \
  --from ghcr.io/anomalyco/opencode:latest \
  --provider openrouter \
  -- opencode -m openrouter/nvidia/nemotron-3.5-lightning:free

openshell sandbox list
openshell sandbox connect agent-0001     # re-attach
openshell sandbox exec -n agent-0001 -- uname -a
openshell sandbox stop|start|delete agent-0001
openshell logs agent-0001
```

The first create is slow, because the driver downloads the images and builds VM disks under
`state_dir`. Free OpenRouter models are slow too (queues and rate limits), but the sandbox itself
adds only milliseconds.

The OpenCode image is a bare Alpine Linux image: it has no `git`, `curl`, `bash`, `node` or
`python3`. Real work needs a custom image (planned under `images/`).

**What the walls look like from inside** (tested on agent-0001):

| Inside the sandbox | Result | Meaning |
|---|---|---|
| `uname -a` | `Linux agent-0001 6.12.76 … aarch64` | Its own kernel. |
| `echo $OPENROUTER_API_KEY` | `openshell:resolve:env:…` | A placeholder, never the real key. |
| `ls /Users` | No such file or directory | No access to the Mac's files. |
| `ip addr` | `lo` only | No network card. |
| `wget https://example.com` | `198.18.0.6 … Permission denied` | DNS answered by the supervisor with a fake address, and the connection is blocked by policy. |

## Rules and policy: audit and approve

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
