---
title: Operations
permalink: /admin/operations/
redirect_from: /operations/
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
| `/var/lib/kevaso/` | The store (`store.db`, or `etcd/` with [the etcd store](#the-store)), the cluster CA (`pki/`), the master key (`kms/`), `audit.log`. Root only. Back it up with `kvoadm backup`: [Backup and restore]({{ '/admin/operations/backup/' | relative_url }}) |
| `/run/kevaso/admin.sock` | The admin socket: root on this host can do anything through it (bootstrap and break-glass) |

**On a worker:**

| Path | What |
|---|---|
| `/var/lib/kevaso-silos/` | VM silos: one directory each (state, cloud-init seed, serial log). The serial log is capped at about 10 MiB, plus one older file, `serial.log.1` |
| `/run/kevaso/` | Host commands run in a directory here. If that directory is removed, it is made again |
| `/var/lib/kevaso-containers/` | Container silos |
| `/var/lib/kevaso-openshell/` | OpenShell gateways and sandbox state |
| `/etc/kevaso-silos.nft` | The silo firewall Kevaso last applied |

**On your computer:**

| Path | What |
|---|---|
| `~/.kevaso/config.yaml` | `kvo` contexts (or `$KVO_CONFIG`), a plain user's certificate under `~/.kevaso/contexts/` |
| `~/.kevaso/clusters/CLUSTER/cluster.yaml` | `kvoadm`'s shared file for the cluster: API, CA, hosts and their addresses, admins' public keys |
| `~/.kevaso/clusters/CLUSTER/local.yaml` | `kvoadm`'s file for this computer: who you are, your SSH key, your admin certificate |
| `~/.kevaso/clusters/CLUSTER/signing.key` | The key `kvoadm` signs `kevasod` and plugins with before installing them. Hosts pin it |
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
| Sign-ins, a refresh token used again after 60 seconds (`refresh-token-reuse`: the session ends), disabled users | `"event"` says which. A reuse within 60 seconds is a race: the session stays, and this log does not record it |
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
| See them, with runtimes and leases | `kvoadm kvo get hosts`, `kvoadm kvo describe host hv2` |
| Take one out for maintenance | `kvoadm host drain hv2`: [Draining a host]({{ '/admin/workers/host-loss/' | relative_url }}) |
| Take one out for good | `kvoadm host remove hv2`: [Removing a host]({{ '/admin/workers/host-loss/#removing-a-host' | relative_url }}) |
| Roll a new `kevasod` over all of them | `kvoadm upgrade --kevasod bin/kevasod-linux-amd64` (and one `--kevasod` per other system and CPU): [High availability]({{ '/admin/operations/ha/' | relative_url }}#upgrade) |
| Put it back | `kvoadm host uncordon hv2` |
| The control-plane host runs silos too | `kvoadm host enable-worker root@HOST --ssh-key … --runtime …` |
| Take Kevaso off a machine | `kvoadm reset root@HOST --ssh-key …`: [Reset a machine]({{ '/admin/workers/#reset-a-machine' | relative_url }}) |

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
[High availability]({{ '/admin/operations/ha/' | relative_url }}) has the rest.

**The first start with etcd moves everything over, once.** `kevasod` copies every object out of
`store.db` into etcd, in order, and then renames the file to `store.db.migrated` and leaves it there.
It logs how many objects it moved (`journalctl -u kevasod`). Revisions carry on from where SQLite
was, so `kvo`, the controllers and the workers just re-list once, by themselves; nobody has to
restart anything.

- **Stopped half-way** (the machine died): the next start wipes what's in etcd and does it again.
- **Done**: it never runs again, even if `store.db.migrated` is still there.
- **A `store.db` next to an etcd that already has its own history** is refused, so one can't
  overwrite the other.
- **There is no switch back.** Take a [backup]({{ '/admin/operations/backup/' | relative_url }}) first. A backup restores onto
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
[High availability]({{ '/admin/operations/ha/' | relative_url }}) page. A member's certificate is renewed
by itself, 30 days before it ends. The API's serving certificate is renewed by itself at two thirds
of its life. Backups are logical (every object, not etcd's own snapshot).
etcd's own clean-up (defragmenting) isn't scheduled.

---

## Keys

| Key | Rotate with |
|---|---|
| Secrets' master key | `kvoadm kms rotate root@HOST --ssh-key …` (refused with more than one system node) |
| A leaked recovery key | `kvoadm kms rotate root@HOST --ssh-key … --new-recovery-key` ([below](#the-master-key-and-the-recovery-key)) |
| A secret's value | `kvo secret set NAME --from-file -` |
| Kevaso's SSH key on your machines | Make a new one ([Installing, step 2]({{ '/admin/setup/prepare/' | relative_url }}#step-2--make-an-ssh-key-for-kevaso)), add its `.pub` to each machine's `authorized_keys`, remove the old line |
| People's access | `kvoadm users disable NAME`: their tokens stop working at once |

### The master key and the recovery key

Values are stored with **envelope encryption**: each value has its own data key, and only data keys
are wrapped by the cluster's master key (a root-only file on the system node). A stolen copy of the
store opens nothing without it.

The **recovery key** that `kvoadm init` printed is the spare key to your secrets: the master keys are
sealed to it in the store, and the system node keeps only its public half, so it can't use it
itself. Keep it offline, with your other recovery codes. Nobody needs it day to day. It's what opens
your [backups]({{ '/admin/operations/backup/' | relative_url }}): a backup without it can't be restored.

| Task | Command |
|---|---|
| New master key (re-wraps data keys; the recovery key is unchanged) | `kvoadm kms rotate root@HOST --ssh-key …` |
| The recovery key leaked: new master key and new recovery key | `kvoadm kms rotate root@HOST --ssh-key … --new-recovery-key` |

After `--new-recovery-key`, the old recovery key opens nothing stored from then on. Copies of the
store made before still open with it and hold the values of then: destroy them, and rotate the
secrets that matter.

---

## Troubleshooting

| Symptom | Look at |
|---|---|
| A silo stays `Pending` | `kvo describe silo NAME`: the `Scheduled` condition says why no host fits (runtime not Ready, no CPU left, labels…) |
| A silo stays `Starting` | Its conditions and init steps in `describe`; on the host, `journalctl -u kevasod` |
| `InitFailed` | `describe` shows the step and the end of its output; `kvo exec` in to look |
| A host is `Unknown` | It hasn't reported for 40 s: is `kevasod` running on it, can it reach the control plane? Its silos keep running if it is only cut off, and Kevaso moves none of them: [When the control plane is lost]({{ '/admin/control-plane-loss/' | relative_url }}) |
| A silo is `Degraded` | `kvo describe silo NAME`: an upkeep task's fix didn't help, and its message is there ([Daemons and upkeep]({{ '/use/silos/daemons/' | relative_url }})) |
| A silo is `Running` but gets no traffic | `kvo describe silo NAME`: the probes table and `Route when`. A silo that isn't ready is held out of its service ([Probes]({{ '/use/silos/probes/' | relative_url }})). Then each port's state: a [leased port]({{ '/use/networking/leases/' | relative_url }}) is closed until it's opened |
| A silo can't reach something | Is there a Grant for it, listed in the silo's `grants`? On OpenShell, also an OpenShellPolicy listed in `openshellPolicies`. `describe` shows a `PolicyApplied` condition with what couldn't be enforced |
| `kvo` says `x509: certificate signed by unknown authority` | You're talking to another cluster than your context's (another machine on the same address or port?) |

If nft refuses a new silo firewall on a host, the last ruleset it took stays in force, and the load
is tried again. `kvoadm kvo describe host` shows `SiloFirewall` false, reason `Rejected`:

```text
nft refused the silo firewall; the last one it took stays in force and the load is retried: …
```

With the worker process down, nothing reloads, so the kernel keeps that last ruleset.

On a host, everything kevasod does is in its journal:

```bash
journalctl -u kevasod -f
```
