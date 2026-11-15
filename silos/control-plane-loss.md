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

---

## Who does what

| | The worker | The control plane | You |
|---|---|---|---|
| **By default** | Keeps every silo running, with its ports and secrets as they were. Deletes nothing. Keeps trying to reach the control plane. | After 40 seconds without a heartbeat, marks the host `Unknown` and each of its silos `HostUnreachable`. Starts no other copy. Places nothing new on that host. | Wait for the host to come back, or, once you know it is off, [move its silos](#moving-a-silo-off-a-host-you-know-is-off) with `--confirm-fenced`. |
| **A silo with `onControlPlaneLoss: Stop`** | Stops that silo (a kill, not a graceful stop) once it has had no contact for the silo's `after`. Other silos on the host are not touched. | Starts it elsewhere on its own only when all four conditions [below](#what-it-buys) hold. | Nothing. |

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
`kvo describe host hv1` (the `Ready` condition, reason `HostLost`). `kvo get hosts` has no state
column.

When the host reports again, `HostUnreachable` turns `False`, reason `HostBack`, and nothing was
stopped or moved.

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

`onHostLoss: Move | Recreate` only describe what a confirmed move does. They never start anything on
their own. The only silos that restart elsewhere by themselves are the ones that [opted in](#the-opt-in-stop-when-the-control-plane-is-lost).

### When that host comes back

It takes the current desired state, like any worker. A silo that was moved away is no longer placed
there, so the host stops its old copy then, because it has now been told to. If the old copy has
volumes on the host's own disk, the stopped copy is **kept** with them, never deleted. A silo that was
not moved simply carries on.

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

`after` counts from the last time the worker reported to the control plane successfully. `kvo get
silos -o wide` shows `stops after 2m` in `ON-CP-LOSS`, and `kvo describe silo` shows the line
described [above](#what-you-see).

### What it buys

The silo is started elsewhere **without a person** only when all four hold:

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

Then the silo's `HostUnreachable` condition turns `False`, reason `RestartedElsewhere`, with the
silence, `after`, margin and how it goes on, and the scheduler places it again. Before that, the
message counts down: `started elsewhere once the control plane has heard nothing from hv1 for
2m16s (16s margin); 1m3s so far`.

The margin exists so the old copy is dead before the new one starts: both sides count from the same
heartbeat, so the worker's count ends first, and the margin covers the fence's own reaction time and
the two clocks running at different rates.

### What it costs

- **The silo goes down whenever its worker is cut off for `after`**, even if nothing else is wrong.
  The stop is a kill, not a shutdown: what is on disk is what a crash leaves.
- **It stays down until contact is back**, or until the control plane starts it elsewhere. When
  contact returns and the silo is still placed on that host, the host starts it again.
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
| Where you see it | The host's `ControlPlaneBehind` condition, status `True`, reason `Held`, in `kvo describe host`. The worker's journal says `HOLD: the control plane is older than what this worker saw` with both numbers. `kvo get hosts` and `kvoadm host list` show nothing for it |
| The condition's message | `the control plane is older than what this worker saw: kept history … revision … (epoch …), seen history … revision … (epoch …). Every silo keeps running as it is and nothing is applied until a deliberate restore (kvoadm restore) starts a new history` |

The hold lasts even when the numbers catch up again. The way out is a deliberate restore of a backup,
`kvoadm init --restore BACKUP` (the message above calls it `kvoadm restore`), which starts a new
history of the same cluster that workers accept: [Backup and restore]({{ '/operations/backup/' | relative_url }}).
A worker also ignores a control plane of a **different cluster** completely, with a log line
(`ignoring a control plane of another cluster`).

---

## What is not built, and what has not run on real machines

- **The hold is not connected to `kevasod worker` yet.** The agent can do it and is tested in-process,
  but the worker does not set it up, so a real worker does not hold today. The watermark is not kept on
  disk by a real worker. The condition `ControlPlaneBehind` is therefore never set on a real host.
- **No command clears a hold.** The only way out is a restore.
- **Not run on real machines under this rule:** the opt-in (stopping, `FenceArmed`, the restart elsewhere), a
  confirmed move after a host went silent, a returning host stopping its old copy, a worker that
  reboots while it cannot reach the control plane.
- **Not built:** a provider powering a host off through its API to give the proof for you. Today the
  proof is you.

See [What's proven where]({{ '/operations/#whats-proven-where' | relative_url }}).
