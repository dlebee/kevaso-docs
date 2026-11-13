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
| A volume on a region class (`block-storage`) | Another host in the same region |
| A volume on a global class (`juicefs`: [JuiceFS]({{ '/providers/juicefs/' | relative_url }})) | Any host that runs the class's node side. On a microVM, shared into it over virtiofs ([Volumes]({{ '/silos/volumes/' | relative_url }})) |
| A container's own files | Not kept: only its volumes count |

The narrowest one wins. See it in the `MOVABLE` column:

```bash
kvo get silos -o wide
```

```text
NAME   RUNTIME            PHASE     READY   HOST      MOVABLE              AGE
api    cloud-hypervisor   Running   yes     hv1       Pinned (hv1)         3d
db-1   docker             Running   yes     cloud-1   Region (ca-central)  3d
runner cloud-hypervisor   Running   yes     hv1       Recreate             1d
dev    cloud-hypervisor   Running   yes     hv1       Anywhere             2h
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
    - { name: data, class: block-storage, size: 200Gi, mountPath: /data }
  availability: { onHostLoss: Move }
```

- **Kevaso checks it when you apply:** `Move` is refused if anything is pinned, and the message
  says what (`disk root uses local on hv1: not movable`).
- **A volume is never left behind:** a volume is always kept data, so `Recreate` (and
  `kvo move --recreate`) is refused for a silo with a host-local volume, with the volume named.
  Keep disposable data on the silo's own disk, put the volume on a class that travels, or use `Wait`.
- **Two copies never run.** A lost host stops its `Recreate` and `Move` silos once it's cut off,
  and the control plane waits until then before starting them elsewhere. `Wait` silos
  are never started elsewhere on their own. [How, and how long it takes](#the-fence).

---

## Moving one yourself

```bash
kvo move silo db-1 --to cloud-2
```

The old host stops the silo and lets go of its volumes first, then the new host starts it.
`kvo describe silo db-1` shows `Moving from` while the old host is still letting go.

| Flag | Meaning |
|---|---|
| `--to HOST` | Where to |
| `--recreate` | A fresh copy from its image instead of its data (only for `onHostLoss: Recreate` silos) |
| `--confirm-fenced` | Its host is lost, and you've made sure the old copy is stopped (powered off, or force-stopped at the provider) |

`kvoadm host drain HOST` hands silos off the same way (the old host stops the silo first, and the new host starts it only after that). It does this only for silos with `onHostLoss: Recreate`, as a fresh copy from the image. Silos with `Wait` stay where they are, and drain says so for each. It does not take `--confirm-fenced`: it is for a host that is still up.

A move from a **lost** host needs `--confirm-fenced`: nobody is there to stop the old copy, so
you say it's stopped. Only say it when the old copy really is stopped: if two copies of a silo
must never run at the same time (a single-writer database, or a system with equivocation penalties),
that is the check that keeps it so.

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
- If the last contact is **older than 30 seconds**, the fence kills this host's `Recreate` and
  `Move` silos (a kill, not a graceful stop). It does this whether the agent is cut off or **not running at all** (`kevasod`
  stopped, or killed). `Wait` silos are left alone.
- A quick restart of `kevasod` stops nothing.

It is `kevasod fence-watch`, with `--fence-after` (default `30s`), `--dir`, `--data` and one
`--runtime` per runtime on the host. You don't run it yourself.

The timing, from the constants in the code:

| | |
|---|---|
| The fence acts | When the last contact is older than **30 s** (checked every second) |
| Every silo it stops is dead | By **36 s** after the last contact |
| The control plane calls the host lost | After **40 s** without a heartbeat |
| It starts the copy elsewhere | **30 s** after that: **67 s** after the last contact at the earliest, allowing for clock skew |

So the old copy is dead at least about 30 seconds before the new one starts. A test of these
constants holds the gap at 15 seconds or more. It has not been measured across two real machines.

What the fence doesn't cover:

- A silo with **no recorded policy** is not stopped. The agent records a silo before it starts it,
  so a running silo without a record is a fault: the fence logs it, and the agent reports it once
  it is back in contact.
- A **control-plane host that is also a worker** (`kvoadm init --worker`, a system node that runs
  silos) has the fence unit, but it acts only when the rest of the control plane can carry on
  without that host (another system node serves the API and the others hold a majority of the
  store). A one-node or two-node control plane has nobody to start the silos elsewhere, so the
  fence stops nothing there.

If you can't live with that, use `Wait`, and when the host really is dead, `kvo move … --confirm-fenced`.
