---
title: Operations
permalink: /operations/
---

# Operating a cluster

The day-two tasks, and where to look when something isn't right.

---

## Where things are

**On a control-plane host:**

| Path | What |
|---|---|
| `/usr/local/bin/kevasod` | The program (installed only after its signature verifies) |
| `/etc/systemd/system/kevasod.service` | Its service |
| `/var/lib/kevaso/` | The store (`store.db`, or `etcd/` with [the etcd store](#the-store)), the cluster CA (`pki/`), the master key (`kms/`), `audit.log`. Root only. Back it up with `kvoadm backup`: [Backup and restore]({{ '/operations/backup/' | relative_url }}) |
| `/run/kevaso/admin.sock` | The admin socket: root on this host can do anything through it (bootstrap and break-glass) |

**On a worker:**

| Path | What |
|---|---|
| `/var/lib/kevaso-silos/` | VM silos: one directory each (state, cloud-init seed, serial log) |
| `/var/lib/kevaso-containers/` | Container silos |
| `/var/lib/kevaso-openshell/` | OpenShell gateways and sandbox state |
| `/etc/kevaso-silos.nft` | The silo firewall Kevaso last applied |

**On your computer:**

| Path | What |
|---|---|
| `~/.kevaso/config.yaml` | `kvo` contexts (or `$KVO_CONFIG`) |
| `~/.kevaso/signing.key` | The key `kvoadm` signs `kevasod` with before installing it |
| macOS Keychain, `com.kevaso.kvo` | Your sign-in tokens |

---

## The audit log

`/var/lib/kevaso/audit.log`, one JSON object per line:

```json
{"time":"…","user":"bob","type":"user","verb":"create","resource":"silos/exec","namespace":"payments","name":"api","allowed":true,"reason":"RoleBinding payments/bob (ClusterRole editor)"}
```

| What's in it | |
|---|---|
| Every decision about a person or service account | Allowed and refused |
| Sign-ins, refresh-token reuse (a stolen token), disabled users | `"event"` says which |
| Every secret value read | `"verb":"read-value"` |
| Every request a silo's local endpoint passes on (a secret, a signature, a leased port) | `"event":"broker"`, as `system:silo:NS:NAME`, via its host |
| Hosts and kevasod itself | Only when refused (they're busy) |

Values are never written to it. It rotates at 100 MiB.

```bash
sudo tail -f /var/lib/kevaso/audit.log | jq -c 'select(.allowed == false)'
```

---

## Hosts

| Task | Command |
|---|---|
| See them, with runtimes and leases | `kvo get hosts`, `kvo describe host hv2` |
| Take one out for maintenance | `kvoadm host drain hv2` |
| Roll a new `kevasod` over all of them | `kvoadm upgrade --kevasod ./kevasod`: [High availability]({{ '/operations/ha/' | relative_url }}#upgrade) |
| Put it back | `kvoadm host uncordon hv2` |
| The control-plane host runs silos too | `kvoadm host enable-worker root@HOST --ssh-key … --runtime …` |

---

## The store

Every object in the cluster (silos, grants, roles, secrets, sealed) lives in the control plane's
**store**. There are two:

| Store | Where | For |
|---|---|---|
| SQLite (default) | One file, `/var/lib/kevaso/store.db` | One control plane |
| etcd | An etcd member inside `kevasod`, in `/var/lib/kevaso/etcd/` | What a control plane of several machines needs (see below) |

### Starting on etcd, or switching

A new cluster starts on etcd with `kvoadm init … --store etcd`. A cluster on SQLite is switched with
one command, once:

```bash
kvoadm store migrate root@203.0.113.10 --ssh-key ~/.ssh/kevaso_ed25519
```

It adds `--store etcd` (and a peer listener) to the service on the control-plane host and restarts it.
[High availability]({{ '/operations/ha/' | relative_url }}) has the rest.

**The first start with etcd moves everything over, once.** `kevasod` copies every object out of
`store.db` into etcd, in order, and then renames the file to `store.db.migrated` and leaves it there.
It logs how many objects it moved (`journalctl -u kevasod`). Revisions carry on from where SQLite
was, so `kvo`, the controllers and the workers just re-list once, by themselves; nobody has to
restart anything.

- **Stopped half-way** (the machine died): the next start wipes what's in etcd and does it again.
- **Done**: it never runs again, even if `store.db.migrated` is still there.
- **A `store.db` next to an etcd that already has its own history** is refused, so one can't
  overwrite the other.
- **There is no switch back.** Take a [backup]({{ '/operations/backup/' | relative_url }}) first. A backup restores onto
  the SQLite store, which is how you go back. Restoring onto a machine that runs etcd moves the
  restored store into etcd on its first start.

### How it's protected

- etcd's client port is `127.0.0.1:2379`: this machine only. Keep it that way
  (`--etcd-client-listen`).
- etcd's members talk to each other and to `kevasod` over mutual TLS, with certificates from the
  cluster's own CA. Only the member's certificate (named `kevaso-etcd`) is let in: a host's or a
  person's certificate from the same CA is refused.
- No other member may connect unless you give `kevasod` a peer address (`--etcd-peer-listen`). Open that port (2380) only between system nodes.

Members of several machines, the light node, losing and removing one, and upgrades are on the
[High availability]({{ '/operations/ha/' | relative_url }}) page. A member's certificate is renewed
by itself, 30 days before it ends. Backups are logical (every object, not etcd's own snapshot).
etcd's own clean-up (defragmenting) isn't scheduled.

---

## Keys

| Key | Rotate with |
|---|---|
| Secrets' master key | `kvoadm kms rotate root@HOST --ssh-key …` (refused with more than one system node) |
| A leaked recovery key | `kvoadm kms rotate root@HOST --ssh-key … --new-recovery-key` ([Secrets]({{ '/secrets/' | relative_url }}#the-master-key-and-the-recovery-key)) |
| A secret's value | `kvo secret set NAME --from-file -` |
| Kevaso's SSH key on your machines | Make a new one ([Installing, step 2]({{ '/setup/prepare/' | relative_url }}#step-2--make-an-ssh-key-for-kevaso)), add its `.pub` to each machine's `authorized_keys`, remove the old line |
| People's access | `kvoadm users disable NAME`: their tokens stop working at once |

---

## What's proven where

Kevaso's tests come in three kinds, and this page says which each feature has had. The source is
`MILESTONES.md` in the core repository.

| Level | Means |
|---|---|
| **In-process** | Unit and integration tests in one program, with fakes for the machines, the cloud and (where noted) the storage |
| **Lima** | The test loop on real Linux VMs on a Mac, through `kvoadm`, with real nftables and (with nested KVM) real microVMs |
| **Linodes** | The same loop on real Linode machines |

| Feature | In-process | Lima | Linodes | Not yet |
|---|---|---|---|---|
| [Addresses, ports and names](../networking/model/): internal and external IPs, `hostPort` per port, `sources`, names, the DNS answers | Yes | | | **A real machine.** The firewall rules for each host address, and the DNS listeners on a host's addresses. A host whose external IP is on no interface (a cloud's 1:1 NAT) isn't handled: `hostPort: [external]` matches nothing there |
| [Port leases](../networking/leases/), per port: several ports opened together, each its own lease and approval | Yes | | | The per-port form on a real machine. The mechanism (cut within a second at the end, `kvo close`, the max, approve and deny, `sources`, from a container) ran on Linodes with one lease for a whole service. Leases on `loadBalancer` ports with a NodeBalancer (the test token had no NodeBalancer access); leases on an Ingress host (not built) |
| Plugin load balancers and `allowFrom` | Yes, against a fake plugin | | | A real NodeBalancer |
| [`IP` kind](../networking/ips/): shared balancers, `dedicated`, `kvo ip keep`, `ip-pool`, the same address for outgoing traffic | Yes, against a fake plugin | | | **A real machine**, and a real provider with this form (a shared IP did answer from the other worker 1 m 13 s after its host went down on Linodes, before the `IP` kind) |
| [Probes and `routeWhen`](../silos/probes/) | Yes | | | **A real machine** |
| [Secrets without anything installed in the silo](../secrets/): injection, files and environment variables written by the host, the local endpoint | Yes | | | **A real machine.** The `curl` lines, the file and env deliveries after a reboot and a reset, the egress certificate in an injecting microVM, a silo without a grant having no endpoint, spoofed source addresses being dropped |
| [Approvals](../secrets/#ask-a-person-first-approvals) | Yes | | | On the local endpoint, on a real machine. The approval flow itself ran on Lima (a microVM and a container, the key never in the silo) with an earlier way for the silo to ask |
| [Private registries](../secrets/): pull secrets | Yes | | | **A real machine**, and a real private registry |
| [Daemons and upkeep](../silos/daemons/) | Yes | Yes: microVM and container | | |
| [Panes](../panes/): the silo serves them, the host fetches them | Yes | | | **A real machine.** (Panes ran on Lima with an earlier way for the silo to report them.) A silo that is cut off and recovers |
| The macOS panes app | Yes: its own 60 tests, against a fake issuer and control plane | | | A run against a real cluster; packaging; the streamed browser |
| [Plugins for `kvo`](../extending/kvo-plugins/) | Yes, with a test plugin, a locally built signed index and a test of the unpacking | | | A published index (none exists yet); launching the panes app through `kvo panes` |
| [HTTPS sites (ingress)](../ingress/): stock proxies driven from outside | Yes | | | **A real machine.** A real nginx started from its own image, HTTPS through it with certificates from a test ACME server over HTTP-01 and DNS-01, a refused setting holding the last good config, a reload on renewal. Caddy and Kong the same |
| [Certificates](../ingress/): ACME and the internal CA | Yes | | | A real machine with the current ingress |
| [Terminals](../silos/terminals/), `kvo cp`, a dropped client | Yes | | Yes (a container silo) | A microVM is not recorded |
| [Moves](../silos/moves/): `kvo move`, `onHostLoss: Move` | Yes | Yes: a container move; a microVM refused as pinned | Yes: a `linode-block` volume moves; `Move` after a power-off; `--confirm-fenced` | |
| [The fence](../silos/moves/#the-fence) | Yes | Yes: `kevasod` stopped or killed; a quick restart stops nothing | | A thicker margin; control-plane hosts that are workers |
| `container:` and `vm:`, [`VMImage`](../silos/sets/) | Yes | | | The blocks as written today on a real machine. Container silos have run on Lima and Linodes (`docker`), microVMs on Lima |
| [Volumes on microVMs](../silos/volumes/): local disks, virtiofs, a root disk on a mount class | Yes, with fakes | | | **A real machine.** Everything about it |
| [JuiceFS](../providers/juicefs/) for container silos | Yes, with the `juicefs` program faked | | Yes: mount, size, move, host power-off, metadata outage | microVMs over virtiofs; a non-root silo; `podman` |
| [The etcd store](#the-store), SQLite to etcd | Yes | | Yes | |
| [Three system nodes, a light node](../operations/ha/): join, lose one, remove one | Yes | | Yes | Removing the leader's own node; five system nodes; Podman workers |
| [`kvoadm upgrade`](../operations/ha/#upgrade) | Yes | | Yes: API up throughout, containers kept running | An upgrade with a light node in the cluster |
| [Backup and restore](../operations/backup/) | Yes | | | A real-machine run is not recorded here |
| [Plugin operators](../extending/plugins/#plugin-operators) | Yes | | Yes: install over SSH and systemd, scale, uninstall | |
| Plugin manifests with privileges | Yes | | Yes: the JuiceFS plugin's | |
| [Scale and `oldSelf`](../extending/kinds/#scale), guards after mutation policies, dry runs | Yes | | (the operator run uses them) | |
| [Linode](../providers/linode/): Block Storage | Yes, against a fake API | | Yes: a volume | NodeBalancer and DNS on real Linode (needs a token with those scopes) |

"Not yet" is a statement about what has been **recorded**, not about whether it works: if it isn't
in the table as proven, assume it isn't.

---

## Troubleshooting

| Symptom | Look at |
|---|---|
| A silo stays `Pending` | `kvo describe silo NAME`: the `Scheduled` condition says why no host fits (runtime not Ready, no CPU left, labels…) |
| A silo stays `Starting` | Its conditions and init steps in `describe`; on the host, `journalctl -u kevasod` |
| `InitFailed` | `describe` shows the step and the end of its output; `kvo exec` in to look |
| A host is `Unknown` | It hasn't reported for 40 s: is `kevasod` running on it, can it reach the control plane? |
| A silo is `Degraded` | `kvo describe silo NAME`: an upkeep task's fix didn't help, and its message is there ([Daemons and upkeep]({{ '/silos/daemons/' | relative_url }})) |
| A silo is `Running` but gets no traffic | `kvo describe silo NAME`: the probes table and `Route when`. A silo that isn't ready is held out of its service ([Probes]({{ '/silos/probes/' | relative_url }})). Then each port's state: a [leased port]({{ '/networking/leases/' | relative_url }}) is closed until it's opened |
| A silo can't reach something | Is there a Grant for it, listed in the silo's `grants`? On OpenShell, `describe` shows a `PolicyApplied` condition with what couldn't be enforced |
| `kvo` says `x509: certificate signed by unknown authority` | You're talking to another cluster than your context's (another machine on the same address or port?) |

On a host, everything kevasod does is in its journal:

```bash
journalctl -u kevasod -f
```
