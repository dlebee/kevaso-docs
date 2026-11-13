---
title: Volumes
permalink: /silos/volumes/
---

# Volumes

A **volume** is storage a silo keeps apart from its image: it survives a reset, and, depending on
its **class**, it can follow the silo to another host. This page is about how volumes work on each
runtime, and what makes a microVM movable.

| Class | Where it lives | The silo can move to |
|---|---|---|
| `local` (default) | On the silo's host | Nowhere: pinned |
| `linode-block` | A Block Storage volume | Another Linode in the same region ([Linode]({{ '/providers/linode/' | relative_url }})) |
| `juicefs` (or any name you give it) | A bucket and a database | Any host that runs the class's node side ([JuiceFS]({{ '/providers/juicefs/' | relative_url }})) |

```yaml
spec:
  volumes:
    - { name: data, class: local, size: 50Gi, mountPath: /data }
    - { name: chain, class: juicefs, size: 500Gi, mountPath: /chain, important: true }
```

| Field | Meaning |
|---|---|
| `name` | The volume's name in the silo |
| `class` | The storage class. Default `local` |
| `size` | How big. See the table below: not every runtime holds it |
| `mountPath` | Where it appears inside the silo |
| `important` | A `Recreate` is refused for a silo with an important local volume |

---

## On containers (`docker`, `podman`)

The host mounts the volume and the container sees it at `mountPath`. `size` isn't enforced.

---

## On microVMs (`cloud-hypervisor`)

A microVM can have volumes too. How one reaches the guest depends on where it lives:

| The volume is… | It reaches the guest as… |
|---|---|
| `local` | A disk of its own (a thin logical volume on the host). The guest formats it once, with ext4, **only if nothing at all is on it**, and mounts it |
| On a class whose storage is a **folder on the host** (a mount class such as `juicefs`, or a device class such as `linode-block` that the host has mounted) | A **virtiofs** share of that folder. The guest never sees a cloud block device |

- **The guest mounts them by itself**, before the guest helper and your init steps run, and again
  on every boot (the lines are in a block of `/etc/fstab` that the host owns, with `nofail`, so a
  missing one never holds a boot).
- **Each shared folder has its own `virtiofsd`**, run as the user `kevaso-vfs`, with no network, no
  device nodes, no set-uid files, and a view of nothing but that folder. Installing the
  `cloud-hypervisor` runtime on a host adds the `virtiofsd` package and the user.
- **Changing the volume list restarts the microVM.** There is no adding a disk to a running one.
- **A volume taken out of the spec keeps its data** until the silo is deleted: put it back and the
  same data is there.
- A **reset** keeps volumes and makes the root disk again.
- `mountPath: /` is refused: a volume can't replace the guest's own disk.
- `size` on a `local` volume of a microVM sets the disk's size when it's made, and **doesn't grow**
  later. On a shared folder it isn't enforced.

> **Not proven on a real machine.** This is built and tested with fakes. It has not yet run on a
> real host: how virtiofs behaves under load, and whether a JuiceFS mount stays visible to
> `virtiofsd` if the plugin's daemons die (a share would keep serving a dead mount until the silo
> restarts).

---

## A microVM that can move

A microVM's own root disk is a thin volume on its host, so by default it is **pinned**. To make it
movable, put the root disk on a **mount class**:

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: dev, namespace: work }
spec:
  runtime: cloud-hypervisor
  image: ubuntu-24.04
  disk: { class: juicefs, size: 40Gi }
  volumes:
    - { name: home, class: juicefs, size: 100Gi, mountPath: /home/dev }
  availability: { onHostLoss: Move }
```

The root disk is then an **image file in a volume of that class** (named `disk.root`), sized
`disk.size`. It is made once, reused when the silo starts on another host, and made again only
by a reset. A silo whose root disk and volumes are all on global classes is movable to
`Anywhere`, and `onHostLoss: Move` and `kvo move` work on it ([Host loss and moves]({{ '/silos/moves/' | relative_url }})).

- **One host at a time.** The control plane points the volume at a single host, and a move waits
  until the old host has stopped the VM and its `virtiofsd` and unmounted the volume before the new
  host mounts it.
- **After a move** the guest takes the new host's key (and only that key) for exec, and has a new
  host key and address.
- **A device class** (`linode-block`) can't hold a root disk yet. It is refused, with the reason.
- `kvo get silos -o wide` shows the result in the `MOVABLE` column.

> **Not proven on a real machine** either: a microVM on a JuiceFS root disk has run only against
> fakes.

---

## Seeing a volume

```bash
kvo describe silo dev -n work
```

has a `Volumes` table: name, class, size (marked `not enforced` when the class can't hold it), mount
path, where it is, and its state (`mounted on hv1`, `attached to hv1`, or the class's message).
