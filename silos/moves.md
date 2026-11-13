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
| A microVM's own disk (`local`) | Nowhere: it's pinned to its host |
| A volume on `local` | Nowhere: pinned |
| A volume on `linode-block` | Another Linode in the same region |
| A volume on a global class (`juicefs`: [JuiceFS]({{ '/providers/juicefs/' | relative_url }})) | Any host that runs the class's node side. Container silos only for now |
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
- **Two copies never run.** A lost host's agent stops its `Recreate` and `Move` silos once it's
  cut off, and the control plane waits until then before starting them elsewhere. `Wait` silos
  are never started elsewhere on their own.

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
