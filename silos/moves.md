---
title: Host loss and moves
permalink: /silos/moves/
---

# Host loss and moves

Silos are stateful, so Kevaso never moves one as a side effect. Two things decide what can
happen when a silo has to leave its host:

1. **What its storage allows.** A silo moves *with its data* only if everything it keeps can
   attach somewhere else.
2. **What the silo wants** when its host is lost: `spec.availability.onHostLoss`.

---

## What can move

| What the silo keeps | Where it can go |
|---|---|
| A microVM's own disk on `local` (the default) | Nowhere: it's pinned to its host |
| A microVM's own disk on a [mount class]({{ '/silos/volumes/#a-microvm-that-can-move' | relative_url }}) (`disk: { class: juicefs }`) | Wherever the class goes: it's an image file on a volume |
| A volume on `local` | Nowhere: pinned |
| A volume on `linode-block` | Another Linode in the same region |
| A volume on a global class (`juicefs`: [JuiceFS]({{ '/providers/juicefs/' | relative_url }})) | Any host that runs the class's node side. On a microVM, shared into it over virtiofs ([Volumes]({{ '/silos/volumes/' | relative_url }})) |
| A container's own files | Not kept: only its volumes count |

The narrowest one wins. See it in the `MOVABLE` column:

```bash
kvo get silos -o wide
```

```text
NAME      RUNTIME            PHASE     HOST   MOVABLE
api       cloud-hypervisor   Running   hv1    Pinned (hv1)
rpc-1     docker             Running   lin1   Region (ca-central)
runner    cloud-hypervisor   Running   hv1    Recreate
dev       cloud-hypervisor   Running   hv1    Anywhere
```

---

## When a host is lost

| `onHostLoss` | What happens |
|---|---|
| `Wait` (default) | The silo stays. It's down until the host comes back. |
| `Recreate` | A fresh copy from its image starts on another host. Its own disk is lost, by declaration. |
| `Move` | It starts on another host **with its data**: its volumes are detached and attached there. |

```yaml
spec:
  runtime: docker
  volumes:
    - { name: chain, class: linode-block, size: 200Gi, mountPath: /data }
  availability: { onHostLoss: Move }
```

- **Kevaso checks it when you apply:** `Move` is refused if anything is pinned, and the message
  says what (`disk root uses local on hv1: not movable`).
- **An important volume is never left behind:** `Recreate` is refused for a silo with a local
  volume marked `important: true`.
- **Two copies never run.** A lost host stops its `Recreate` and `Move` silos once it's cut off,
  and the control plane waits until then before starting them elsewhere. `Wait` silos
  are never started elsewhere on their own. [How, and how long it takes](#the-fence).

---

## Moving one yourself

```bash
kvo move silo rpc-1 --to lin2
```

The old host stops the silo and lets go of its volumes first, then the new host starts it.
`kvo describe silo rpc-1` shows which step it's on.

| Flag | Meaning |
|---|---|
| `--to HOST` | Where to |
| `--recreate` | A fresh copy from its image instead of its data (only for `onHostLoss: Recreate` silos) |
| `--confirm-fenced` | Its host is lost, and you've made sure the old copy is stopped (powered off, or force-stopped at the provider) |

A move from a **lost** host needs `--confirm-fenced`: nobody is there to stop the old copy, so
you say it's stopped. For a validator, that's what keeps two copies from signing the same slot.

Moving needs `create` on `silos/move` (the built-in `editor` and `admin` roles have it).

---

## The fence

A lost host may not be lost at all: it may only be cut off, with its silos still running and
writing. So before the control plane starts a copy elsewhere, the host has to stop its own.

Every worker host runs a second, small service for this, `kevaso-fence.service`, separate from
`kevasod` and installed by `kvoadm host add --install worker`:

```bash
sudo systemctl status kevaso-fence
```

- The agent writes a record every second to `/run/kevaso-fence/`: when it last reached the control
  plane, and each silo's `onHostLoss`.
- If the last contact is **older than 30 seconds**, the fence stops this host's `Recreate` and
  `Move` silos. It does this whether the agent is cut off or **not running at all** (`kevasod`
  stopped, or killed). `Wait` silos are left alone.
- A quick restart of `kevasod` stops nothing.

It is `kevasod fence-watch`, with `--fence-after` (default `30s`), `--dir`, `--data` and one
`--runtime` per runtime on the host. You don't run it yourself.

The timing, measured on Lima machines with a container silo:

| | |
|---|---|
| The control plane calls the host lost | After **40 s** without a heartbeat |
| It starts the copy elsewhere | At least **30 s** after that: 70 s |
| The fence stops the old copy | Within about **60 s** of the last contact |

So the old copy is stopped about ten seconds before the new one starts. **That margin is thin.** On a
test, a copy was stopped 46 to 48 seconds after `kevasod` stopped, and the new one started at
about 66 to 69.

What the fence doesn't cover:

- A silo with **no recorded policy** is treated as `Wait`. The record comes from what the agent
  has synced, so a silo created while the agent was down isn't covered.
- A **control-plane host that is also a worker** (`kvoadm init --worker`, a system node that runs
  silos) has no fence unit.
- The fence stops a container with Docker's own 20-second grace; it is not a forced stop.

If you can't live with that, use `Wait`, and when the host really is dead, `kvo move … --confirm-fenced`.
