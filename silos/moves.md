---
title: Moves
permalink: /silos/moves/
---

# Moves

Silos are stateful, so Kevaso never moves one as a side effect. Two things decide what can
happen when a silo has to leave its host:

1. **What its storage allows.** A silo moves *with its data* only if everything it keeps can
   attach somewhere else.
2. **What the silo wants** once its old host is known to be off: `spec.availability.onHostLoss`.

What happens when a host is simply out of reach is on another page:
[When the control plane is lost]({{ '/silos/control-plane-loss/' | relative_url }}).

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
NAME     RUNTIME            PHASE     READY   HOST      MOVABLE               ON-CP-LOSS   AGE
api      cloud-hypervisor   Running   yes     hv1       Pinned (hv1)          -            3d
db-1     docker             Running   yes     cloud-1   Region (ca-central)   -            3d
runner   cloud-hypervisor   Running   yes     hv1       Recreate              -            1d
dev      cloud-hypervisor   Running   yes     hv1       Anywhere              -            2h
```

---

## What `onHostLoss` means

| `onHostLoss` | What a confirmed move does |
|---|---|
| `Wait` (default) | The silo stays on its host. It can still be moved by hand if its storage can move |
| `Recreate` | A fresh copy from its image starts on another host. Its own disk is lost, by declaration |
| `Move` | It starts on another host **with its data**: its volumes are detached and attached there |

`onHostLoss` never moves a silo on its own. It says how a move goes once the old host is known to be
off: you run `kvo move … --confirm-fenced`, or the silo [opted in]({{ '/silos/control-plane-loss/#the-opt-in-stop-when-the-control-plane-is-lost' | relative_url }}) to be
stopped and restarted elsewhere.

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
- **`exclusive: true`** requires `Wait`: a move is by hand, once the old copy is known to be stopped.

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
that is the check that keeps it so. [How to use it, and what a returning host does]({{ '/silos/control-plane-loss/#moving-a-silo-off-a-host-you-know-is-off' | relative_url }}).

`--confirm-fenced` is refused while the source host claimed a start of this silo less than 30
seconds ago. The host may count as lost, but it answered then and may be starting the silo, so the
old copy is not provably stopped. That keeps a second copy from starting. The message names the
host, how long ago, and how long to wait. For a claim 10 seconds old it is:

```text
kvo: Conflict (409): move payments/api: its host just claimed its start of this silo: baremetal-1 claimed its start 10s ago, so it answered then and may be starting it now: it is not provably stopped. Wait 20s, or add --insist if you know the old copy is off
```

Wait for the time it names. `kvo move` has no `--insist` flag. Adding one is refused:

```text
kvo: unknown flag --insist
```

Moving needs `create` on `silos/move` (the built-in `editor` and `admin` roles have it).
