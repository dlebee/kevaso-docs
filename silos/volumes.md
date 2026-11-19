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
| `block-storage` | A cloud block volume | Another host in the same region ([provider page]({{ '/providers/linode/' | relative_url }})) |
| `juicefs` (or any name you give it) | A bucket and a database | Any host that runs the class's node side ([JuiceFS]({{ '/providers/juicefs/' | relative_url }})) |

```yaml
spec:
  volumes:
    - { name: data, class: local, size: 50Gi, mountPath: /data }
    - { name: archive, class: juicefs, size: 500Gi, mountPath: /archive }
```

| Field | Meaning |
|---|---|
| `name` | The volume's name in the silo |
| `class` | The storage class. Default `local` |
| `size` | How big. See the table below: not every runtime holds it |
| `mountPath` | Where it appears inside the silo |

A volume has no retention flag: it is always **kept data**. Disposable data (a cache, scratch
space) belongs on the silo's own disk, not on a volume.

A provider volume taken out of the spec is detached once the host has unmounted it, and kept until
the silo is deleted. Put the name back and the same volume is there. Changing a volume's class in
place is refused, and so is `disk.class` on a microVM. Empty and `local` are the same class.

```text
volume data: class can't change in place (local to juicefs); delete the silo to drop the volume
```

```text
disk.class can't change in place (local to juicefs); delete the silo to drop the disk
```

- A silo whose volumes are on a host-local class can't be recreated elsewhere. `onHostLoss:
  Recreate` and `kvo move --recreate` are refused while the silo has such a volume, naming it:
  a fresh copy elsewhere would leave the volume's data behind. Put the volume on a class that
  travels, keep the disposable data on the silo's own disk, or use `Wait`.
- If a silo is placed on another host while it still has a `local` volume, the old host stops the
  copy and **keeps it** with its volumes. It removes the copy only when the silo is deleted. If the silo
  is placed back there, the copy is reused with its volumes.

---

## What Kevaso refuses

**A name too long for the provider's label.** A volume on a class from a provider plugin (and a
microVM's root disk on such a class) gets a label on the provider's side:
`kvo-NAMESPACE-SILO-VOLUME`, then `-` and the first five characters of the silo's ID. A provider
allows 32 characters, so `kvo-NAMESPACE-SILO-VOLUME` may be at most **26**. A longer one is **refused,
never cut**: a cut label would give two volumes the same name, and a provider that finds a volume by
label would hand one of them the other's disk. A silo set is checked with its longest instance name.
`local` volumes have no provider label and are not checked.

```text
Silo payments: volume db-data: the provider's label kvo-production-payments-db-data is 31 characters and the label keeps 26, so volumes would share one: shorten the namespace, the silo's or the volume's name by 5
```

It is refused when you apply (status 422). An update that adds no volume is not checked again.

**A `local` volume larger than the host has left.** On a microVM, a `local` volume is a thin volume in
the host's `kevaso` volume group. The host never promises more than the pool holds: before it makes
a volume it adds up what the pool already promised, and refuses one that doesn't fit. The silo does not
start. A volume with no `size` counts as `10Gi`.

```text
volume data needs 40Gi: this host's storage (kevaso/silos) is 100Gi, 70Gi of it promised to other disks already (30Gi left)
```

This is the host's own check, made when the volume is created. The scheduler does not look at storage,
so it does not choose a host with room: pick one with `placement`. It covers microVM volumes only;
containers' `local` volumes and volumes on provider classes are not checked this way. Running out of
space *inside* a volume is the silo's own problem.

**A device path outside `/dev/disk/by-id/`.** The host mounts a device-class volume only from a name
under `/dev/disk/by-id/`. Anything else is refused, and the silo does not start:

```text
not mounting /dev/sdb: a volume's device is under /dev/disk/by-id/
```

---

## On containers (`docker`, `podman`)

The host mounts the volume and the container sees it at `mountPath`. `size` isn't enforced.

---

## On microVMs (`cloud-hypervisor`)

A microVM can have volumes too. How one reaches the guest depends on where it lives:

| The volume is… | It reaches the guest as… |
|---|---|
| `local` | A disk of its own (a thin logical volume on the host). The guest formats it once, with ext4, **only if nothing at all is on it**, and mounts it |
| On a class whose storage is a **folder on the host** (a mount class such as `juicefs`, or a device class such as `block-storage` that the host has mounted) | A **virtiofs** share of that folder. The guest never sees a cloud block device |

- **The guest mounts them by itself**, before your init steps run, and again
  on every boot (the lines are in a block of `/etc/fstab` that the host owns, with `nofail`, so a
  missing one never holds a boot).
- **Each shared folder has its own `virtiofsd`**, in a locked-down unit of its own: no network, no
  device nodes, and once it is serving it sees nothing but that folder. It runs as root but holds
  only the capabilities it needs to act for the guest's users; it can't create device nodes, set-uid
  files or file capabilities. Installing the `cloud-hypervisor` runtime on a host adds the
  `virtiofsd` package.
- **Changing the volume list restarts the microVM.** There is no adding a disk to a running one.
- **A volume taken out of the spec keeps its data** until the silo is deleted: put it back and the
  same data is there. A provider volume is detached only after this host has unmounted it, and the
  volume itself stays until the silo is deleted.
- A **reset** keeps volumes and makes the root disk again.
- `mountPath: /` is refused: a volume can't replace the guest's own disk.
- `size` on a `local` volume of a microVM sets the disk's size when it's made, and **doesn't grow**
  later. On a shared folder it isn't enforced.

> **Proven only on virtual machines.** Local volumes and a mount class's volume over virtiofs have
> run on Lima machines with a test provider plugin. Not proven: virtiofs under load, a JuiceFS
> volume under a microVM, and whether a JuiceFS mount stays visible to `virtiofsd` if the plugin's
> daemons die (a share would keep serving a dead mount until the silo restarts).

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
  vm: { image: ubuntu-24.04 }
  disk: { class: juicefs, size: 40Gi }
  volumes:
    - { name: home, class: juicefs, size: 100Gi, mountPath: /home/dev }
  availability: { onHostLoss: Move }
```

The root disk is then an **image file in a volume of that class** (named `disk.root`), sized
`disk.size`. It is made once, reused when the silo starts on another host, and made again only
by a reset. That volume stays mounted for the life of the placement: while the silo is placed
here, the host does not unmount `disk.root`, including while the copy looks absent. A move
unmounts it only after the old host has stopped the VM.

A silo whose root disk and volumes are all on global classes is movable to
`Anywhere`, and `onHostLoss: Move` and `kvo move` work on it ([Moves]({{ '/silos/moves/' | relative_url }})).

- **One host at a time.** The control plane points the volume at a single host, and a move waits
  until the old host has stopped the VM and its `virtiofsd` and unmounted the volume before the new
  host mounts it.
- **After a move** the guest takes the new host's key (and only that key) for exec, and has a new
  host key and address.
- **A device class** (`block-storage`) can't hold a root disk yet. It is refused: a VM's disk can live on a mount class, not a device class.
- `kvo get silos -o wide` shows the result in the `MOVABLE` column.

> **Proven on shared network storage between two Lima machines, not on JuiceFS.** Two moves, and a
> `Move` after the old host was cut off, kept one copy running and kept the files. A microVM on a
> JuiceFS root disk has not run.

---

## Seeing a volume

```bash
kvo describe silo dev -n work
```

has a `Volumes` table: name, class, size (marked `not enforced` when the class can't hold it), mount
path, where it is, and its state (`mounted on hv1`, `attached to hv1`, or the class's message).
