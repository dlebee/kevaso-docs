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
| Take one out for maintenance | `kvoadm host drain hv2`: [Draining a host]({{ '/workers/host-loss/' | relative_url }}) |
| Roll a new `kevasod` over all of them | `kvoadm upgrade --kevasod ./kevasod`: [High availability]({{ '/operations/ha/' | relative_url }}#upgrade) |
| Put it back | `kvoadm host uncordon hv2` |
| The control-plane host runs silos too | `kvoadm host enable-worker root@HOST --ssh-key … --runtime …` |
| Take Kevaso off a machine | `kvoadm reset root@HOST --ssh-key …`: [Reset a machine]({{ '/workers/#reset-a-machine' | relative_url }}) |

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

This table says which test level each feature has reached. The sources are `MILESTONES.md` and
`test/e2e/METAL.md` in the core repository.

| Level | Means |
|---|---|
| **In-process** | Unit and integration tests in one program, with fakes for the machines, the cloud and (where noted) the storage |
| **Local VMs** | The test loop on real Linux VMs on a Mac, through `kvoadm`, with real nftables and (with nested KVM) real microVMs |
| **Cloud machines** | The same loop on rented cloud machines. **These runs were made before the sealed addressing model and the removal of the guest helper**, so they show what worked then, not what works now |
| **Bare metal, one machine** | 13 end-to-end tests on one dedicated machine with a real external address, KVM and Ubuntu 26.04 (2026-11-13): addresses and ports, names, images, secrets on microVMs and containers, ingress with a certificate, probes, leases, panes, the fence on one machine, volumes, a private registry, Podman, a reboot. Every test passed, and the machine was clean again after each run |
| **Bare metal, two machines** | The two-machine core path, ten tests (2026-11-13), a system node and a worker over their external addresses: the worker alone, the mesh, moves, the fence, a worker rebooted while it holds silos, ingress in front of a silo on the other machine, a drain, an upgrade, silo storage on a whole disk of the second machine, and the reset of it. All ten passed, under the rules of that day: the fence and the host-loss tests were rewritten for the current rule on 2026-11-14 and have not run since |

**Not yet, for anything:** the failure cases (killed mid-start, an interrupted download, bad images, a cut that is shorter than the fence, a reboot in the middle of a move, a move between unlike storage) and a long soak are written and have not run. **Not yet on real machines**, because they are newer than the last run: a worker that is cut off and keeps its silos; the opt-in `onControlPlaneLoss: Stop` stopping and restarting a silo elsewhere; `kvo move … --confirm-fenced` after a host went silent; a cluster started with `kvoadm init` and a node refusing to start with its data gone; a worker holding; silos kept when a restore does not know them; `--storage-device` in its current form; one broken Ingress left out while the rest are served; the bridge-port and IPv6 checks. OpenShell has not run on bare metal.

| Feature | In-process | Local VMs | Cloud machines (before the sealed model) | Bare metal | Not yet |
|---|---|---|---|---|---|
| [Addresses, ports and names]({{ '/networking/model/' | relative_url }}): internal and external IPs, `hostPort` per port, `sources`, names, the DNS answers | Yes |  |  | Yes: ports on the external IP, `sources`, names and the DNS answers. An `[internal]` port is refused, with the reason, on a machine with no internal address | A machine with an internal address. A host whose external IP is on no interface (a cloud's 1:1 NAT): it doesn't work there. IPv6 isn't used |
| [Port leases]({{ '/networking/leases/' | relative_url }}), per port | Yes | Yes: per port, on a microVM silo | Yes, with one lease for a whole service (before per-port leases) | Yes: closed until leased, answers while open, a held connection is cut when the lease ends, approval, `kvo close` | Leases on `loadBalancer` ports with a provider's balancer. Leases on an Ingress host: not built |
| Plugin load balancers and `allowFrom` | Yes, against a fake plugin |  |  |  | A real balancer |
| [`IP` kind]({{ '/networking/ips/' | relative_url }}): shared balancers, `dedicated`, `kvo ip keep`, `ip-pool`, the same address for outgoing traffic | Yes, against a fake plugin |  | A shared address answered from the other worker 1 m 13 s after its host went down (before the `IP` kind) |  | A real machine with the `IP` kind, and a real provider with this form |
| [Probes and `routeWhen`]({{ '/silos/probes/' | relative_url }}) | Yes |  |  | Yes: an external port and the service's name are held until the probe passes, and again when it fails |  |
| [Secrets with nothing installed in the silo]({{ '/secrets/' | relative_url }}): injection, files and variables written by the host, the local endpoint | Yes |  |  | Yes: on a microVM and on Docker (including Kevaso's runtime keeping a variable out of `docker inspect` and Docker's disk), injection, the endpoint | Files and variables on Podman. The fallback for variables on a real host. OpenShell |
| [Approvals]({{ '/secrets/#ask-a-person-first-approvals' | relative_url }}) | Yes | Yes, with an earlier way for the silo to ask |  | Yes: through the silo's local endpoint, and for a leased port |  |
| [Private registries]({{ '/secrets/#private-registry-credentials' | relative_url }}): pull secrets | Yes |  |  | Yes: against a registry on the machine itself, over plain HTTP | A registry with TLS, or on another host |
| [Daemons and upkeep]({{ '/silos/daemons/' | relative_url }}) | Yes | Yes: microVM and container |  |  |  |
| [Panes]({{ '/panes/' | relative_url }}): the silo serves them, the host fetches them | Yes | Yes, with an earlier way for the silo to report them |  | Yes: microVM and container, and a silo that answers too much is cut off | A silo that is cut off and recovers |
| The macOS panes app | Yes: its own 60 tests, against a fake issuer and control plane |  |  |  | A run against a real cluster; packaging; the streamed browser |
| [Plugins for `kvo`]({{ '/extending/kvo-plugins/' | relative_url }}) | Yes, with a test plugin, a locally built signed index and a test of the unpacking |  |  |  | A published index (none exists yet); launching the panes app through `kvo panes` |
| [HTTPS sites (ingress)]({{ '/ingress/' | relative_url }}): stock proxies driven from outside | Yes, for nginx, Caddy and Kong |  | Yes, for nginx, before the sealed model | Yes: nginx on its own image, a certificate from Let's Encrypt's staging directory over HTTP-01, the redirect to HTTPS, unknown hosts answered 404 | Caddy and Kong on a real machine. Leaving out one broken Ingress while the others are applied, and the search for it. A reload on renewal. Let's Encrypt production. A site whose silo is on another machine |
| [Certificates]({{ '/ingress/' | relative_url }}): ACME and the internal CA | Yes |  |  | Yes: ACME over HTTP-01 against the staging directory | DNS-01 on a real machine. The internal CA on a real machine |
| [Roles can't give more than their author holds]({{ '/access/#you-cant-give-more-than-you-hold' | relative_url }}) | Yes: about fifty cases, through the API |  |  |  | A real cluster |
| [`sources` checked and IPv6 refused]({{ '/networking/model/#sources-and-ipv6' | relative_url }}); a silo sends only from its own address | Yes: properties over every source address and frame, a fuzzer over the rules (550,000 runs), IPv6 from every side |  |  |  | The checks on a real host (written, not run) |
| [Terminals]({{ '/silos/terminals/' | relative_url }}), `kvo cp`, a dropped client | Yes |  | Yes (a container silo) |  | A microVM is not recorded |
| [Moves]({{ '/silos/moves/' | relative_url }}): `kvo move`, `onHostLoss: Move` | Yes | Yes: a container move; a microVM refused as pinned | Yes: a block volume moves; `Move` after a power-off; `--confirm-fenced` | Two machines: a container moves with its data and leaves nothing behind; a microVM with a local volume is refused | A reboot in the middle of a move; a move between unlike storage (written) |
| [`kvoadm host drain`]({{ '/workers/host-loss/' | relative_url }}): the hand-off | Yes |  |  | Two machines | Ending a hand-off whose old host is lost, on real machines |
| [A worker and the control plane lose each other]({{ '/silos/control-plane-loss/' | relative_url }}): nothing changes, `HostUnreachable`, `kvo move … --confirm-fenced`, the returning host stops its old copy | Yes: every combination of policy and silence, on a fake clock; the real controllers; a lost host and a returning one | Yes, under the earlier rule: the fence with `kevasod` stopped or killed | Yes, under the earlier rule: `Move` after a power-off, `--confirm-fenced` | Two machines, under the earlier rule only (the fence and the host-loss tests) | The current rule on real machines |
| [The opt-in `onControlPlaneLoss: Stop`]({{ '/silos/control-plane-loss/#the-opt-in-stop-when-the-control-plane-is-lost' | relative_url }}): the fence, `FenceArmed`, the restart elsewhere after `after` plus a margin | Yes: the margin and every missing piece (not armed, one second short, an older spec, another host); the fence's record and timing; the real controllers with the host's clock an hour behind |  |  | The fence on one machine: a quick restart stops nothing, and with the control plane on the same machine the record says there is nothing to fence | Stopping and restarting elsewhere on two machines; a thicker margin; control-plane hosts that are workers |
| [A worker holding]({{ '/silos/control-plane-loss/#a-control-plane-older-than-what-the-worker-last-saw' | relative_url }}) a control plane older than what it saw | Yes: the judgement table, and the agent keeping everything | | | | **Not connected to `kevasod worker`**, so a real worker doesn't hold; real machines |
| [Starting a cluster]({{ '/operations/backup/#starting-a-cluster-and-a-node-that-finds-its-data-gone' | relative_url }}): `kvoadm init` creates it, a node with its data gone refuses | Yes: SQLite, etcd, joined nodes, a restore | | | | `kvoadm init`'s create step and the refusals on a real machine; a restore announced to real workers |
| Silos the control plane doesn't know are kept | Yes: a silo missing from the API is kept; a delete the agent saw removes it | | | | A real run (written) |
| `container:` and `vm:`, [`VMImage`]({{ '/silos/sets/' | relative_url }}) | Yes |  | Container silos with `docker`, in the earlier form | Yes: a container image by tag and by digest, the built-in microVM image, an inline disk with its hash, a wrong hash refused with the reason |  |
| [Volumes on microVMs]({{ '/silos/volumes/' | relative_url }}): local disks, names too long refused, a volume larger than the host's storage refused | Yes, with fakes |  |  | Yes: data survives a guest reboot, a stop and start, and `kvo reset` of the root disk; deleting the silo removes the volume | virtiofs and a root disk on a mount class: nothing but in-process |
| [JuiceFS]({{ '/providers/juicefs/' | relative_url }}) for container silos | Yes, with the `juicefs` program faked |  | Yes: mount, size, move, host power-off, metadata outage |  | microVMs over virtiofs; a non-root silo; `podman`; the sealed model |
| `podman` as the container engine | Yes |  |  | Yes: a silo runs, opens an external port, and gets a secret from the local endpoint; then it is taken off again | Files and variables on Podman |
| A reboot of the machine | Yes |  |  | Yes: Kevaso, the fence, the runtimes, the silos and a microVM's volume come back |  |
| [`kvoadm reset`]({{ '/workers/#reset-a-machine' | relative_url }}), [`--storage-device`]({{ '/workers/#where-silo-disks-live' | relative_url }}) | Yes: its plan and its checklist; the disk check as a pure function and the runtime on a fake host |  |  | Yes: after each run, the machine is clean again; the reset of a named disk, with and without `--wipe-storage` | A machine whose install has no record; `--storage-device` in its current form |
| [The etcd store]({{ '#the-store' | relative_url }}), SQLite to etcd | Yes |  | Yes |  |  |
| [Three system nodes, a light node]({{ '/operations/ha/' | relative_url }}): join, lose one, remove one | Yes |  | Yes |  | Removing the leader's own node; five system nodes; Podman workers |
| [`kvoadm upgrade`]({{ '/operations/ha/#upgrade' | relative_url }}) | Yes |  | Yes: API up throughout, containers kept running |  | An upgrade with a light node in the cluster |
| [Backup and restore]({{ '/operations/backup/' | relative_url }}) | Yes |  |  |  | A real-machine run is not recorded |
| [Plugin operators]({{ '/extending/plugins/#plugin-operators' | relative_url }}) | Yes |  | Yes: install over SSH and systemd, scale, uninstall |  |  |
| Plugin manifests with privileges | Yes |  | Yes: the JuiceFS plugin's |  |  |
| [Scale and `oldSelf`]({{ '/extending/kinds/#scale' | relative_url }}), guards after mutation policies, dry runs | Yes |  | (the operator run uses them) |  |  |
| [Provider plugins]({{ '/providers/linode/' | relative_url }}): block storage | Yes, against a fake API |  | Yes: a volume |  | Load balancer and DNS against a real provider (needs a token with those scopes) |

"Not yet" is a statement about what has been **recorded**, not about whether it works: if it isn't
in the table as proven, assume it isn't.

---

## Troubleshooting

| Symptom | Look at |
|---|---|
| A silo stays `Pending` | `kvo describe silo NAME`: the `Scheduled` condition says why no host fits (runtime not Ready, no CPU left, labels…) |
| A silo stays `Starting` | Its conditions and init steps in `describe`; on the host, `journalctl -u kevasod` |
| `InitFailed` | `describe` shows the step and the end of its output; `kvo exec` in to look |
| A host is `Unknown` | It hasn't reported for 40 s: is `kevasod` running on it, can it reach the control plane? Its silos keep running if it is only cut off, and Kevaso moves none of them: [When the control plane is lost]({{ '/silos/control-plane-loss/' | relative_url }}) |
| A silo is `Degraded` | `kvo describe silo NAME`: an upkeep task's fix didn't help, and its message is there ([Daemons and upkeep]({{ '/silos/daemons/' | relative_url }})) |
| A silo is `Running` but gets no traffic | `kvo describe silo NAME`: the probes table and `Route when`. A silo that isn't ready is held out of its service ([Probes]({{ '/silos/probes/' | relative_url }})). Then each port's state: a [leased port]({{ '/networking/leases/' | relative_url }}) is closed until it's opened |
| A silo can't reach something | Is there a Grant for it, listed in the silo's `grants`? On OpenShell, `describe` shows a `PolicyApplied` condition with what couldn't be enforced |
| `kvo` says `x509: certificate signed by unknown authority` | You're talking to another cluster than your context's (another machine on the same address or port?) |

On a host, everything kevasod does is in its journal:

```bash
journalctl -u kevasod -f
```
