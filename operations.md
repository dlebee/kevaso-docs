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
| Every broker call from a silo | `"event":"broker"`, as `system:silo:NS:NAME`, via its host |
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
| [Port leases](../networking/leases/) on `hostPort` | Yes | | Yes: cut within a second at the end, `kvo close`, the max, approve and deny, `lease.allowFrom`, from a container | Leases on `loadBalancer` classes with a NodeBalancer (the test token had no NodeBalancer access); leases on an Ingress host (not built) |
| Plugin load balancers and `allowFrom` | Yes, against a fake plugin | | | A real NodeBalancer |
| [Approvals](../secrets/#ask-a-person-first-approvals) | Yes | Yes: a microVM and a container; the key never in the silo | | |
| [Daemons and upkeep](../silos/daemons/) | Yes | Yes: microVM and container | | |
| [Panes](../panes/) (status, HTML, bundle) | Yes | Yes: microVM and container | | |
| The macOS panes app | Yes: its own 60 tests, against a fake issuer and control plane | | | A run against a real cluster; packaging; the streamed browser |
| [Terminals](../silos/terminals/), `kvo cp`, a dropped client | Yes | | Yes (a container silo) | A microVM is not recorded |
| [Moves](../silos/moves/): `kvo move`, `onHostLoss: Move` | Yes | Yes: a container move; a microVM refused as pinned | Yes: a `linode-block` volume moves; `Move` after a power-off; `--confirm-fenced` | |
| [The fence](../silos/moves/#the-fence) | Yes | Yes: `kevasod` stopped or killed; a quick restart stops nothing | | A thicker margin; control-plane hosts that are workers |
| [Volumes on microVMs](../silos/volumes/): local disks, virtiofs, a root disk on a mount class | Yes, with fakes | | | **A real machine.** Everything about it |
| [JuiceFS](../providers/juicefs/) | Yes, with the `juicefs` program faked | | | **A real machine.** In progress |
| [The etcd store](#the-store), SQLite to etcd | Yes | | Yes | |
| [Three system nodes, a light node](../operations/ha/): join, lose one, remove one | Yes | | Yes | Removing the leader's own node; five system nodes; Podman workers |
| [`kvoadm upgrade`](../operations/ha/#upgrade) | Yes | | Yes: API up throughout, containers kept running | An upgrade with a light node in the cluster |
| [Plugin operators](../extending/plugins/#plugin-operators) | Yes | | Yes: install over SSH and systemd, scale, uninstall | |
| Plugin manifests with privileges | Yes | | | A privileged plugin on a real host |
| [Scale and `oldSelf`](../extending/kinds/#scale), guards after mutation policies, dry runs | Yes | | (the operator run uses them) | |
| [Linode](../providers/linode/): Block Storage, shared IP | Yes, against a fake API | | Yes: a volume; a shared IP that answered from the other worker 1 m 13 s after its host went down | NodeBalancer and DNS on real Linode (needs a token with those scopes) |

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
| A silo can't reach something | Is there a Grant for it, listed in the silo's `grants`? On OpenShell, `describe` shows a `PolicyApplied` condition with what couldn't be enforced |
| `kvo` says `x509: certificate signed by unknown authority` | You're talking to another cluster than your context's (another machine on the same address or port?) |

On a host, everything kevasod does is in its journal:

```bash
journalctl -u kevasod -f
```
