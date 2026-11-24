---
title: When the control plane is lost
permalink: /silos/control-plane-loss/
---

# When a worker and the control plane lose each other

**By default, nothing changes.** A worker that can't reach the control plane keeps running every silo
it was told to run, and the control plane never starts a second copy of a silo on another host by
itself.

A cut-off worker and a dead one look the same from the control plane. A cut-off worker is still
running its silos, so a second copy elsewhere could be two writers of the same data, or the same
identity in two places. Kevaso does not guess. A person who knows the host is off says so, or a silo
asks, in advance, to be stopped when it is cut off.

A silo is never started on two hosts. Before a host starts a silo, it records a start claim on that
silo. While the claim is fresh, the control plane leaves the silo on that host. That keeps a second
copy from starting while the first host may still be starting it.

---

## Who does what

| | The worker | The control plane | You |
|---|---|---|---|
| **By default** | Keeps every silo running, with its ports as they were. An injected secret keeps going for up to one hour after the control plane last gave it, then is refused until the control plane answers. A revoked grant or a rotated secret is still injected for that hour. A file or env value already in the silo stays. Grants have no lifetime of their own yet, so the window is always one hour. Deletes nothing. Keeps trying to reach the control plane. | After 40 seconds without a heartbeat, marks the host `Unknown` and each of its silos `HostUnreachable`. Starts no other copy. Places nothing new on that host. | Wait for the host to come back, or, once you know it is off, [move its silos](#moving-a-silo-off-a-host-you-know-is-off) with `--confirm-fenced`. |
| **A silo with `onControlPlaneLoss: Stop`** | Stops that silo (a kill, not a graceful stop) once it has had no contact for the silo's `after`. Other silos on the host are not touched. | Starts it elsewhere on its own only when all five conditions [below](#what-it-buys) hold. The fifth is that its host reported the stop. | Nothing. |

What a worker cannot do while it is cut off is change anything: no new silo, no `kvo apply`, no new
secret. Silos keep what they already have.

---

## What you see

A silent host's silos stay assigned to it. In `kvo get silos -o wide` their phase is `Unknown`, and
the new `ON-CP-LOSS` column shows which silos would stop themselves:

```bash
kvo get silos -o wide
```

```text
NAME     RUNTIME            PHASE     READY   HOST   MOVABLE        ON-CP-LOSS       AGE
api      cloud-hypervisor   Unknown   yes     hv1    Pinned (hv1)   -                3d
signer   docker             Unknown   yes     hv1    Anywhere       stops after 2m   3d
```

`READY` is what the host last reported, so it can still say `yes`. Go by `PHASE` and by the
`HostUnreachable` condition.

```bash
kvo describe silo api -n payments
```

The `Conditions` table has a row `HostUnreachable`, status `True`, reason `Waiting`. Its message says
since when the host has been silent, that no other copy is started, and what you can do. It is one
of these, depending on the silo:

| The silo | The message ends with |
|---|---|
| `onHostLoss: Recreate` | `Once the old copy is known to be off (the host powered off, or force-stopped through its provider), kvo move silo NAME -n NAMESPACE --to HOST --confirm-fenced --recreate starts a fresh copy from its image (onHostLoss: Recreate; data on its own disk is lost)` |
| Storage that can move | `Once the old copy is known to be off (…), kvo move silo NAME -n NAMESPACE --to HOST --confirm-fenced moves it with its data (onHostLoss: Move), or wait for hv1 to come back` |
| Pinned to the host | `Waiting for hv1 to come back (onHostLoss: Wait; it can't move: …; a fresh copy needs onHostLoss: Recreate and kvo move … --confirm-fenced --recreate)` |

All three start with `hv1 unreachable since 2026-11-14T20:02:11Z. It may still be running there, so
Kevaso starts no other copy.`, with the host's name and the time it went silent.

`kvo describe silo` also has a line `On control-plane loss`: `keeps running (onControlPlaneLoss:
Continue)`, or `STOPS: the host stops this silo after 2m without the control plane
(onControlPlaneLoss: Stop)`. A silo set shows the same for its instances.

The host itself shows as `Unknown` in `kvoadm host list` (`Unknown (no heartbeat for 1m3s)`) and in
`kvoadm kvo describe host hv1` (the `Ready` condition, reason `HostLost`). `kvoadm kvo get hosts` has no state
column.

When the host reports again, `HostUnreachable` turns `False`, reason `HostBack`, and nothing was
stopped or moved.

### Deleting a silo while its host is unreachable

`kvo delete silo` removes the object at once when nothing is holding it. The name can be used
again. The copy on the unreachable host keeps running until that host sees the delete. Creating
the same name before the host is back can run a second copy under that name. Wait for the host,
or confirm the machine is off
([below](#moving-a-silo-off-a-host-you-know-is-off), or
[remove the host]({{ '/workers/host-loss/#removing-a-host' | relative_url }}) with `--confirm-off`),
before you create it again.

A silo with a plugin volume keeps the object until that volume's finalizer is gone
(`kevaso.com/volumes`). The name stays taken for that time. A local disk has no such finalizer.

---

## Moving a silo off a host you know is off

Make sure the old copy is stopped: power the machine off, or force-stop it through its provider.
Then say so:

```bash
kvo move silo api -n payments --to hv2 --confirm-fenced
```

For a silo with `onHostLoss: Recreate`, which starts a fresh copy from its image:

```bash
kvo move silo runner -n payments --to hv2 --confirm-fenced --recreate
```

| Without `--confirm-fenced`, for a lost host | |
|---|---|
| The refusal | `its host is lost: move it only once the old copy is provably stopped (host powered off, or force-stopped through the provider), then add --confirm-fenced` |

- **Who may say it.** `--confirm-fenced` and `--insist` need `create` on `hosts/confirm-off`.
  That is cluster-scoped, and only `cluster-admin` has it by default. A plain move does not need
  it. The 403, and how to grant it, are on [Moves]({{ '/silos/moves/#moving-one-yourself' | relative_url }}).
- **Only say it when it is true.** `--confirm-fenced` is the one place where Kevaso takes your word. If
  the old copy is still running, you now have two.
- **`onHostLoss` says how.** `Move` starts it elsewhere with its data (the volumes are detached from
  the lost host and attached to the new one). `Recreate` starts a fresh copy from the image and gives up
  the data on the silo's own disk. A silo with `Wait` that can move with its data moves with its data;
  a pinned one cannot move at all. [Moves]({{ '/silos/moves/' | relative_url }}).
- **A hand-off that was under way** (`kvoadm host drain`, or `kvo move`) when the old host went
  silent is ended the same way: the same command with `--confirm-fenced`.
- **Before the 40 seconds are up** the host still counts as up, and a move is the ordinary hand-off:
  the new host waits until the old one has stopped the silo and let go.
- **A fresh start claim refuses `--confirm-fenced`.** If that host claimed a start of this silo
  less than 30 seconds ago, the move is refused even when the host otherwise counts as lost. It
  answered then and may be starting the silo, so the old copy is not provably stopped. Wait for the
  time the message names. [The message]({{ '/silos/moves/#moving-one-yourself' | relative_url }}).

`onHostLoss: Move | Recreate` only describe what a confirmed move does. They never start anything on
their own. The only silos that restart elsewhere by themselves are the ones that [opted in](#the-opt-in-stop-when-the-control-plane-is-lost).

### When that host comes back

It takes the current desired state, like any worker. A silo that was moved away is no longer placed
there, so the host stops its old copy then, because it has now been told to. If the old copy has
volumes on the host's own disk, the stopped copy is **kept** with them, never deleted. A silo that was
not moved simply carries on.

A worker that was down still sees a silo deleted while it was away, when it comes back. It resumes
its delete watch from the last revision it stored, so that delete is in the log and the copy is
removed, disk included. If the store's log no longer goes back that far, or the stored revision
cannot be read, the watch starts at the list it has now. The copy deleted in the gap is **kept**,
running, with its disk and volumes. It is listed for `kvoadm host kept`
([below](#a-control-plane-older-than-what-the-worker-last-saw), and on
[Backup and restore]({{ '/operations/backup/#what-workers-do-after-a-restore' | relative_url }})).
The journal says:

```text
delete watch: revision 1200 was compacted; a delete since then was not seen, and that copy is kept
```

An unreadable record says `starting at the list's revision, so a delete while this host was away is not seen and that copy is kept`.

---

## The opt-in: stop when the control plane is lost

For the narrow case where running twice does more harm than not running at all, a silo can ask its
worker to stop it when the control plane has been out of reach for a while. It is off unless you write
it on the silo. Nothing else turns it on: not `onHostLoss`, not `exclusive`, not a storage class.

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: signer, namespace: payments }
spec:
  runtime: docker
  container: { image: registry.example.com/signer:1.4 }
  availability:
    onHostLoss: Recreate
    onControlPlaneLoss: Stop
    after: 2m
```

It is meant for a system with equivocation penalties, a single-writer database that keeps no lock of
its own, or any silo whose second copy is worse than its absence.

### The fields

They are under `spec.availability`. A silo set carries them in `spec.template.availability`, so every
instance has them.

| Field | Values | Default | Limits |
|---|---|---|---|
| `onControlPlaneLoss` | `Continue`, `Stop` | `Continue` | Anything else is refused: `availability.onControlPlaneLoss must be Continue or Stop` |
| `after` | A duration, `2m`, `90s` | `2m` with `Stop` | At least `30s`, at most `24h`. Only with `Stop` |

| Refused | Message |
|---|---|
| `after` out of range or not a duration | `availability.after "10s": want a duration from 30s to 24h0m0s (e.g. 2m)` |
| `after` without `Stop` | `availability.after goes with onControlPlaneLoss: Stop (Continue changes nothing, so there's nothing to wait for)` |
| `Stop` on `openshell` or `openshell-container` | `onControlPlaneLoss: Stop isn't available on openshell: its gateway brings its sandboxes back by itself (a restart of the gateway, a reboot), so nothing keeps it stopped without the control plane` (the runtime's own name is in the message) |

`after` counts from the last time the worker reported to the control plane successfully. `kvo get
silos -o wide` shows `stops after 2m` in `ON-CP-LOSS`, and `kvo describe silo` shows the line
described [above](#what-you-see).

Applying a silo on `openshell` or `openshell-container` with `Stop` is refused, with the message in
the table (`Silo NAME: ` in front). The gateway brings that sandbox back on its own, so nothing can
keep it stopped. A silo set with that template is not refused when you apply the set. Each instance
is refused, with the same message, when it is created.

### What it buys

The silo is started elsewhere **without a person** only when all five hold:

1. The silo has `onControlPlaneLoss: Stop`.
2. `onHostLoss` is `Recreate`, or `Move` on storage that can move. With `Wait` (and so with
   [`exclusive`]({{ '/silos/sets/#exclusive-one-copy-ever' | relative_url }}), which requires `Wait`) the silo stops itself and stays
   assigned to its host, which waits. The `HostUnreachable` message says so.
3. The host's worker confirmed that its fence stops this silo, for this version of the spec: the silo
   has a condition `FenceArmed`, status `True`, reason `StopsAfter`, for its current host and its
   current `after`. If it never did, the message says `onControlPlaneLoss: Stop, but hv1 never
   confirmed its fence stops this silo (condition FenceArmed for generation N): not started elsewhere`.
4. The control plane has heard nothing from that host for `after` plus a margin, counted on its own
   clock. The margin is 15 seconds plus a thousandth of `after`, rounded up to a second: `16s` for
   `2m`, `1m42s` for `24h`.
5. Its host reported that the fence stopped its copy. The report is the silo condition `FenceStopped`,
   reason `StoppedByFence`. The host writes it when it can reach the control plane again. It counts
   only for that host's latest start of the silo. Silence, however long, is never enough.

Then the silo's `HostUnreachable` condition turns `False`, reason `RestartedElsewhere`, with the
silence, `after`, margin and how it goes on, and the control plane starts it elsewhere. Without the
host's report, the condition says the silo waits for its host to confirm the stop, and it names
`kvo move silo NAME -n NS --to HOST --confirm-fenced`. Add `--recreate` when needed.

The host's report is what shows the old copy is stopped before a new one starts. A fence kill can
fail, and then the old copy may still be running.

### What it costs

- **The silo goes down whenever its worker is cut off for `after`**, even if nothing else is wrong.
  The stop is a kill, not a shutdown: what is on disk is what a crash leaves.
- **It stays down during a real cut.** Nothing restarts until its host reports the stop, or you run
  `kvo move silo NAME -n NS --to HOST --confirm-fenced`. Add `--recreate` when needed. When contact
  returns and the silo is still placed on that host, the host starts it again.
- **Recreate gives up the silo's own disk**, and `Move` needs every disk and volume on storage that
  can move ([Moves]({{ '/silos/moves/' | relative_url }})).
- **Every Stop silo needs the fence.** `kevaso-fence.service` is installed on every worker by
  `kvoadm`. Without it the host cannot confirm the fence, and nothing restarts elsewhere.

### When not to use it

**Do not set it where the control plane is not always reachable.** That is one system node, or a
control plane on a machine that is sometimes off or travels with a laptop. The silo stops every time
the control plane is out of reach for `after`: a reboot, an upgrade, a network blip, a closed lid. The
rule prevents a silo that stops for a reason that has nothing to do with a second copy.

The same goes for two system nodes without a light node, where losing either stops the store
([High availability]({{ '/operations/ha/' | relative_url }})). Use it where the control plane
survives the loss of a machine.

Kevaso prints a warning when you apply a silo, or a silo set, with `Stop` on a cluster that has
exactly one system node (a light, state-only node does not count). It does not refuse:

```text
Warning: Silo signer: onControlPlaneLoss: Stop, and this cluster has exactly one system node: the silo will stop whenever that node is unreachable (a reboot, a network blip, a control plane on a laptop that travels). Use it only where the control plane is always reachable, i.e. with more than one system node
```

If you cannot live with two copies and cannot guarantee the control plane, use the default and move
the silo yourself with `--confirm-fenced`.

---

## The fence

The fence is how a worker stops a Stop silo when `kevasod` itself is not running. It is a second,
small service on every worker, `kevaso-fence.service`, separate from `kevasod` and installed by
`kvoadm host add --install worker` and by `kvoadm init --worker`:

```bash
sudo systemctl status kevaso-fence
```

- The agent writes a record every second to `/run/kevaso-fence/`: when it last reported to the
  control plane, and for each silo on the host the `after` it asked for (none if it keeps running).
- When the last contact is older than a silo's `after`, the fence kills that silo. It does this
  whether the agent is cut off or **not running at all**. Silos with no `after` are left alone.
- A quick restart of `kevasod` stops nothing, since `after` is at least 30 seconds.
- **On a host with no Stop silo the fence does nothing**: the record says so (`fence: false`).

It is `kevasod fence-watch`, with `--dir`, `--data` and one `--runtime` per runtime on the host. It
also takes `--fence-after`, which has no effect on records that carry each silo's own `after`, which
is every record the agent writes. You don't run it yourself.

The silo's own host counts on a one-node cluster too: a Stop silo stops there whenever it is cut off.
That is why the warning exists.

---

## A control plane older than what the worker last saw

A worker remembers which cluster it belongs to and the highest change number it has seen from its
control plane. The store numbers every change, so a number is not a clock that drifts. If a control
plane answers with a **lower** number than the worker already saw (a disk snapshot put back, a copy of
the data directory, a system node serving old data), the worker does not trust it. It **holds**.

| A held worker | |
|---|---|
| Silos | Keep running, all of them, as they are |
| Deletes | Nothing: no silo, volume, secret file, lease or route is removed because the control plane no longer lists it |
| New desired state | None applied from that control plane |
| Heartbeats | Keeps sending them |
| Where you see it | The host's `ControlPlaneBehind` condition, status `True`, reason `Held`, in `kvoadm kvo describe host`. The worker's journal says `HOLD: the control plane is older than what this worker saw (kept: history … revision … epoch …; seen: history … revision … epoch …): keeping everything as it is`. `kvoadm kvo get hosts` and `kvoadm host list` show nothing for it |
| The condition's message | `the control plane is older than what this worker saw: kept history … revision … (epoch …), seen history … revision … (epoch …). Every silo keeps running as it is and nothing is applied until a deliberate restore (kvoadm init --restore) starts a new history, or a person who checked the control plane runs kvoadm host trust-control-plane [USER@]ADDRESS for this host` |
| The local endpoint (`169.254.42.1`) | Answers 503: `this host is held (its control plane is older than what it saw): nothing is asked of it until that ends` |
| Injected secrets | A secret this host was given before the hold keeps being injected for up to one hour, including a grant revoked or a secret rotated while the host is cut off. After that, and for a secret it was not given before the hold, injection is refused until the hold ends. A file or env value already in the silo stays |

A hold within one history does not end when a later answer of that same history reaches the revision
the worker kept. It ends in one of two ways. A deliberate restore,
`kvoadm init --restore BACKUP --no-hosts-added` (or `--host-added NAME` for each host joined since
the backup), starts a new history of the same cluster, and workers accept it. They do not start a
copy that is not already running until that silo is settled
([Backup and restore]({{ '/operations/backup/' | relative_url }})). Or you run
`kvoadm host trust-control-plane` after you have checked that the control plane this host will reach
is the newest of this cluster, and that every silo this host runs is placed here in it. The
condition's message names both. When the hold ends, the journal says
`the control plane is trusted again (…): looking at every silo`.

A silo this host still has, whose delete was never seen here, keeps running with its disk and
volumes when a new history doesn't know it. `kvoadm kvo describe host` lists those under `KeptSilos`.
The same SSH flags as `trust-control-plane` apply.

```bash
kvoadm host kept root@baremetal-1
```

```bash
kvoadm host kept root@baremetal-1 --remove demo/web/0123abcd
```

With no `--remove` it lists them, or prints `baremetal-1 keeps no copy the control plane doesn't know`.
`--remove` takes `NAMESPACE/NAME/UID` (the first 8 characters of the UID are enough). It asks you to
type the silo's name (`demo/web`). `--yes` skips that question. Removing it deletes that copy, its
disk and its volumes. The control plane doesn't know it, so nothing else holds this data.

A worker also ignores a control plane of a **different cluster** completely, with a log line
(`ignoring a control plane of another cluster`).

### An unreadable record

The worker keeps the record in `/var/lib/kevaso/control-plane-watermark.json`, under its data
directory. If that file is there and cannot be read, the worker holds as well. It cannot tell
whether the control plane it reaches is older than one it saw, so it starts, removes and applies
nothing. A healthy answer does not clear this hold.

| Where | What it shows |
|---|---|
| `kvoadm kvo describe host` | `ControlPlaneBehind`, status `True`, reason `WatermarkUnreadable` |
| The condition's message | `this worker's record of its control plane (PATH) is unreadable (WHY): it can't tell whether this control plane is older than one it saw. Every silo keeps running as it is; nothing is started, removed or applied meanwhile. Check that this host's control plane is the one it last used (not one restored from an older backup outside kvoadm init --restore), then run kvoadm host trust-control-plane [USER@]ADDRESS (the address of HOST)` |
| The worker's journal | `CONTROL-PLANE WATERMARK UNREADABLE: PATH: WHY. This worker keeps running every silo and starts, removes and applies nothing until a person checks its control plane and runs kvoadm host trust-control-plane for this host` |

`WHY` is the read error, `not a watermark: …`, or `it names no cluster`.

If the worker cannot save that record, it holds the same way. `kvoadm kvo describe host` shows
`ControlPlaneBehind`, status `True`, reason `WatermarkNotSaved`. The message names the path and the
error, and says every silo keeps running and nothing is started, removed or applied until the record
can be saved. Check the disk on that host. The hold ends once a save succeeds.

Check that the control plane this host will reach is the newest of this cluster, then:

```bash
kvoadm host trust-control-plane root@baremetal-1
```

| Flag | Meaning |
|---|---|
| `[USER@]ADDRESS` | The worker. The user defaults to `root` |
| `--ssh-key PATH` | The key to log in with. Without it, the command uses your key from the cluster folder's `local.yaml`, else `kvoadm`'s own key in that folder. If it does not: `--ssh-key PATH: kvoadm logs in with a key (no password logins)` |
| `--host-key SHA256:…` | The machine's host key. Without it, you are asked to confirm the fingerprint |
| `--ssh-port N` | The machine's SSH port (default 22) |
| `--data DIR` | The worker's data directory (default `/var/lib/kevaso`). A clean absolute path |
| `--yes` | Don't ask |

| The file | What the command does |
|---|---|
| Missing | Prints `→ baremetal-1 has no record of a control plane: nothing to do (it takes the first one it reaches)` and changes nothing |
| Readable, and not held | Refuses, and changes nothing: `/var/lib/kevaso/control-plane-watermark.json on baremetal-1 is readable and not held: nothing to set aside (kevasod reads an unreadable record again by itself every second, so that hold ends once the file reads; the Host's ControlPlaneBehind says so)` |
| Readable, and held | Sets the file aside, the same way as an unreadable one. It says the control plane answered older than the revision this worker saw, and that a hold within one history ends only by a new history (`kvoadm init --restore`) or by this command |
| Unreadable | Prints the lines below, keeps the file as `PATH.unreadable-TIME`, restarts `kevasod`, and prints `→ set aside; kevasod restarted on baremetal-1: it trusts the control plane it reaches next` |

```text
/var/lib/kevaso/control-plane-watermark.json on baremetal-1 is unreadable (it names no cluster).
kvoadm will keep it as /var/lib/kevaso/control-plane-watermark.json.unreadable-TIME and restart kevasod: the worker then trusts the control plane it reaches next as its own.
Do this only once you know that control plane is the newest of this cluster (not one restored from an older backup outside kvoadm init --restore), and that every silo this host runs is placed here in it.
Type yes to go on:
```

Without `--yes` it asks `Type yes to go on: `. Anything else is `not confirmed: nothing was changed`.

A hold whose record can still be read ends when you run this command, or when a restore starts a
new history. A later answer of the same history does not end it. A record that is readable and not
held is left alone.

---

## What is not built, and what has not run on real machines

- **Run on bare metal (2026-11-15):** the fence; an opted-in Stop silo moved with
  `kvo move … --confirm-fenced` after its host was cut or shut down; a held worker serving its last
  state, the hold ended with `kvoadm host trust-control-plane`.
- **Not run on real machines under this rule:** the opt-in stopping and then restarting a silo
  elsewhere by itself after the host reports `FenceStopped`; a cut-off worker keeping its silos; a
  returning host stopping its old copy; a worker that reboots while it cannot reach the control plane;
  a hold ended by a new history from `kvoadm init --restore`.
- **Not built:** a provider powering a host off through its API to give the proof for you. Today the
  proof is you.

See [What's proven where]({{ '/operations/#whats-proven-where' | relative_url }}).
