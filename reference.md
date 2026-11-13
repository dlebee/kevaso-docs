---
title: Command reference
permalink: /reference/
---

# Command reference

Every command on one page. `kvo` uses your current context (`kvo ctx`); `-n NS` picks a namespace
(default: the context's).

---

## kvo: using the platform

### Objects

| Command | Does |
|---|---|
| `kvo apply -f FILE` (or `-f -`) | Create or update every object in a YAML/JSON file |
| `kvo get KIND [NAME] [-n NS \| -A] [-o yaml\|json]` | List or show objects |
| `kvo describe KIND NAME` | Everything about one object, for people |
| `kvo delete KIND NAME` | Delete it |
| `kvo watch KIND [-n NS \| -A]` | Print the current state, then every change |

Kinds: `namespaces` (`ns`), `hosts`, `silos`, `silosets` (`ss`), `services` (`svc`), `images`
(`img`), `secrets`, `grants`, `ippools`, `ingresses` (`ing`), `ingressclasses`, `certificates`
(`cert`), `issuers`, `roles`, `rolebindings`, `clusterroles`, `clusterrolebindings`,
`serviceaccounts` (`sa`), `approvals`, `panes`, `kinddefinitions` (`kd`), `validationpolicies`,
`mutationpolicies`, and any [kind you define]({{ '/extending/kinds/' | relative_url }}) (by its plural,
or `PLURAL.GROUP` when two share one).

| Command | Does |
|---|---|
| `kvo get kinds` | Every kind the cluster serves, built-in and defined, with its group and version |

### Silos

| Command | Does |
|---|---|
| `kvo exec silo NAME -- CMD [ARGS…]` | Run a command inside; exits with its code; stdin passes through |
| `kvo exec -it silo NAME -- CMD [ARGS…]` | The same on a terminal (`-t`; `-i` is accepted and changes nothing). [Terminals]({{ '/silos/terminals/' | relative_url }}) |
| `kvo ssh silo NAME` | A login shell on a terminal (`bash -l` or `sh -l`). Not SSH: the same channel as `exec` |
| `kvo cp SRC DST [--max-size SIZE]` | Copy a file or folder in or out. One side is `silo/NAME:/ABSOLUTE/PATH`. Default limit `1GiB` |
| `kvo ui silo NAME [--no-browser]` | A web page on `127.0.0.1` with the silo's terminal and files. Ctrl-C stops it |
| `kvo reset silo NAME` | Back onto its image: stopped, a fresh disk or container, volumes kept (a set's instance: onto the set's current image) |
| `kvo protect silo NAME [--off]` | A set's scale-in never removes it |
| `kvo move silo NAME --to HOST` | Move it with its data (`--recreate`: a fresh copy; `--confirm-fenced`: from a lost host) |
| `kvo get silos -o wide` | Adds `MOVABLE`: where each silo could go |

### Panes

| Command | Does |
|---|---|
| `kvo get panes [-n NS \| -A]` | The panes silos report: `SILO`, `PANE`, `KIND`, `STATE`, `UPDATED` |
| `kvo pane watch SILO` | Print a silo's panes, then every change, as plain text |
| `kvo pane bundle SILO.PANE -o FILE` | Save an HTML pane's page to a file (never printed) |

See [Panes]({{ '/panes/' | relative_url }}).

### Sets

| Command | Does |
|---|---|
| `kvo scale siloset NAME --instances N` | Grow or shrink; shrinking lists the instances it deletes and needs `--yes` |
| `kvo reset siloset NAME --rolling` | Reset every instance onto the current image, a few at a time |

### Secrets

| Command | Does |
|---|---|
| `kvo create secret NAME --from-file F\|-` | Store a value (never from the command line) |
| `kvo create secret NAME --generate [--type ed25519\|ecdsa-p256]` | Let Kevaso make it |
| …`--allow MODES`, `--write-only`, `-l KEY=VALUE` | Restrict delivery, make write-only, label |
| `kvo secret set NAME --from-file F\|-` | A new value (rotation) |
| `kvo get secret NAME --reveal` | Print the value (needs `read-value`; never for write-only) |
| `kvo get approvals` / `kvo watch approvals` | Silos waiting for a person's yes to use a secret |
| `kvo approve NAME` | Say yes (needs `approve` on that secret; people only) |
| `kvo deny NAME [--reason R]` | Say no |

### Access

| Command | Does |
|---|---|
| `kvo login SERVER --ca sha256:…` | Sign in (browser + invite code) |
| `kvo logout` | Forget this context's session |
| `kvo ctx [NAME]` | List contexts, or switch |
| `kvo ns [NAME]` | Show or set the default namespace |
| `kvo auth whoami` | Who you are to the cluster |
| `kvo auth can-i VERB RESOURCE[/NAME] [-n NS]` | Exits 0 for yes, 1 for no |
| `kvo create token SA [--duration 1h]` | A service-account token, for CI (`$KVO_TOKEN`) |

### Environment

| Variable | Means |
|---|---|
| `KVO_CONFIG` | The config file (default `~/.kevaso/config.yaml`) |
| `KVO_TOKEN`, `KVO_CA_FILE` | A service-account token and the cluster CA, for CI |
| `KVO_SERVER` | A plain-HTTP server (`kevasod dev`, local experiments) |

---

## kvo-guest: inside a silo

The silo's helper for talking to its host. In microVM silos it's installed on first boot; in a
container silo only if the image has it. [Secrets]({{ '/secrets/' | relative_url }}) lists its other commands.

| Command | Does |
|---|---|
| `kvo-guest pane report NAME [--state S] [--progress P] [--title T] [--message M] [--field LABEL=VALUE]… [--ttl D]` | Report a status pane. `S`: `idle`, `running`, `needs-you`, `done`, `failed`. `P`: 0 to 1. `D`: up to `1h`. Prints `ok` |
| `kvo-guest pane html NAME --bundle FILE [--data JSON \| --data -] [--title T] [--ttl D]` | Report an HTML pane: uploads the page, then reports it. The silo needs `spec.panes.allow: [status, html]` |

Both need `spec.panes.enabled: true` on the silo. [Panes]({{ '/panes/' | relative_url }}) has the limits.

---

## kvoadm: running the platform

### Machines

| Command | Does |
|---|---|
| `kvoadm init [USER@]ADDR --ssh-key KEY --host-key SHA256:… --kevasod BIN [--name N] [--cluster C] [--api-server URL]` | Install the control plane; write your kvo context; print the recovery key |
| `kvoadm host enable-worker [USER@]ADDR --ssh-key KEY [--runtime R]… [--loop-storage SIZE]` | The control-plane host runs silos too |
| `kvoadm host reconnect [USER@]ADDR --ssh-key KEY [--server URL]` | Point a worker at the control plane again (after a restore onto another address) |
| `kvoadm host add [USER@]ADDR --ssh-key KEY --kevasod BIN --install worker [--name N] [--region R] [--label K=V] [--runtime R] [--behind-nat]` | Add a worker |
| `kvoadm host list` | Hosts and their state |
| `kvoadm host drain NAME` / `uncordon NAME` | Maintenance |

Every command that logs in takes `--ssh-key PATH` (a key with a passphrase is taken from `ssh-agent`
or asked for) or `--password` (typed once, read from stdin when there's no terminal; kvoadm then adds
a key of its own), `--ssh-port N`, and `--host-key SHA256:…` (without it, you're asked to confirm
the fingerprint).

`init` and `host add` also take the extra hardening, each only when asked for:
`--disable-password-login` (SSH keys only) and `--change-root-password` (a random one nobody
knows). In a terminal without these flags they ask; `--no-extra-hardening` skips the questions.
Everything that asks has a flag, so every command runs unattended too.

Both look at the machine first and stop, changing nothing, if something won't work there;
`--dry-run` stops after looking.

### People

| Command | Does |
|---|---|
| `kvoadm users invite NAME [--namespace NS [--role editor\|admin\|viewer]] [--group G]…` | A one-time sign-in code |
| `kvoadm users list` | Everyone who can sign in |
| `kvoadm users disable NAME` | Cut someone off now |

### Keys

| Command | Does |
|---|---|
| `kvoadm kms rotate [USER@]ADDR --ssh-key KEY` | A new master key for secrets |
| …`--new-recovery-key` | Also replace the recovery key (it leaked) |

### Backups

| Command | Does |
|---|---|
| `kvoadm backup [USER@]ADDR --ssh-key KEY --output FILE` | A backup of the control plane, sealed to the recovery key |
| `kvoadm init [USER@]ADDR … --restore FILE` | A fresh machine from a backup; asks for the recovery key (or reads it from stdin), then reconnects the workers |
| `kvoadm plugin install [USER@]ADDR --name N --binary FILE [--credential NAME=FILE]… [--config FILE] [--side controller\|node]` | A provider plugin as a hardened service (credentials read from files, root-only on the host) |
| `kvoadm plugin remove [USER@]ADDR --name N` | Removes it: unit, program, credentials |

---

## kevasod: on the hosts

You rarely run it yourself; `kvoadm` installs it as a service.

| Command | Does |
|---|---|
| `kevasod control-plane --name N --address A [--worker --runtime R…]` | The control plane (and worker) |
| `kevasod worker --name N --server URL` | A worker-only host |
| `kevasod admin recovery-key` / `kms-rotate [--new-recovery-key]` / `backup` | Through the admin socket, as root |
| `kevasod admin restore [--data DIR]` | Put a backup into a fresh data directory (`kvoadm init --restore` runs it) |
| `kevasod control-plane --store sqlite\|etcd` | Where objects live: SQLite (default) or an embedded etcd member. The first `etcd` start moves `store.db` into it. [Operations]({{ '/operations/#the-store' | relative_url }}) |
| `kevasod control-plane --etcd-client-listen ADDR` | With `--store etcd`: the member's client address (default `127.0.0.1:2379`; keep it on loopback) |
| `kevasod control-plane --etcd-peer-listen ADDR` / `--etcd-advertise-peer ADDR` | With `--store etcd`: where other members connect, and what they dial. Without the first, no peer port is opened |
| `kevasod control-plane --plugins-dir DIR` | Where provider plugins' sockets are (default `/run/kevaso-plugins`) |
| `kevasod control-plane --cert-controller N=PROGRAM` / `--dns-provider N=PROGRAM` | Register programs (also picked up from `/etc/kevaso/cert-controllers/` and `/etc/kevaso/dns-providers/`) |
| `kevasod dev` | The API without authentication on loopback, for trying `kvo` locally |
