---
title: Adding workers
permalink: /workers/
---

# Adding workers

One control plane, any number of workers: a server in a data center, a VPS, a box at home behind
your router. They don't need to be on the same network.

---

## How workers connect

- A new worker **joins with a one-time token** at `/enroll` and gets a certificate for exactly its
  own name and the role `worker`. `/enroll` issues only a worker certificate. A system node joins
  on its own path ([High availability]({{ '/operations/ha/' | relative_url }}#add-a-system-node)).
  It can't act as another host, and it can't touch silos placed elsewhere.
- It **dials out** to the control plane and keeps that connection open; `kvo exec` and everything
  else come back through it. Nothing needs to listen on the worker.
- Hosts are on a **WireGuard mesh** run by the control plane. Each host gets its own silo subnet
  (`10.64.N.0/24`), so silos on different hosts reach each other by address and name.
- A worker **behind NAT** reaches the others through a relay (a host that can be reached directly),
  end to end encrypted inside it: the relay only ever sees encrypted UDP.

The certificate lasts 30 days. The worker renews it by itself at two thirds of that life (or inside
the last ten days, if that comes sooner) and presents the new one on the next connection. A renewal
is the same host. A renewal that is interrupted keeps the key that matches the certificate. A
failure is tried again, from 10 seconds, doubling up to 5 minutes.

An expired certificate is not used again. If a join token is still on the machine, the worker sets
the old folder aside (`/var/lib/kevaso/pki.expired-DATE`) and enrolls again. With no token it stops:

```text
host hv2 certificate in /var/lib/kevaso/pki has expired. Join again with kvoadm host add
```

Add it again, the same way as [after a removal]({{ '/workers/host-loss/#the-removed-machine' | relative_url }}):

```bash
bin/kvoadm host add root@198.51.100.20 \
    --ssh-key ~/.ssh/kevaso_ed25519 \
    --host-key SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4 \
    --kevasod bin/kevasod-linux-amd64 \
    --install worker \
    --name hv2
```

Your own `kvo` certificate renews the same way. When it has expired, see
[Contexts]({{ '/access/#contexts' | relative_url }}).

---

## Prepare the machine

The same as for the system node: [Prepare a machine]({{ '/setup/prepare/' | relative_url }}). Put your
Kevaso key on it and write down its fingerprint. `host add` looks at the machine before it creates
anything (`--dry-run` to only look).

---

## Add it

```bash
bin/kvoadm host add root@198.51.100.20 \
    --ssh-key ~/.ssh/kevaso_ed25519 \
    --host-key SHA256:… \
    --kevasod bin/kevasod-linux-amd64 \
    --install worker \
    --name hv2 \
    --region ca-east \
    --runtime cloud-hypervisor
```

| Flag | Meaning |
|---|---|
| `--install worker` | This host only runs silos |
| `--internal-ip IP`, `--external-ip IP` | The machine's addresses, as for `kvoadm init` (neither: the SSH address, as internal IP) |
| `--region`, `--zone` | Where it is, for placement (`placement.region` and `placement.zone` on a silo) |
| `--label k=v` | Any label, for `placement.hostSelector` |
| `--runtime` | Which runtimes to install (repeatable) |
| `--behind-nat` | Nothing can reach it from outside: it'll use the relay |
| `--server URL` | The API address *as this host reaches it*, if not the one in your context |
| `--egress-extra-ca PATH` | A PEM file on the machine. The egress gate also trusts it for the servers silos connect to. Written into the worker's unit |
| `--openshell-gateway-scope host\|namespace\|silo` | How many OpenShell gateways this worker runs. Written into the worker's unit. If you leave it out, the worker uses `namespace` (one gateway per namespace) |

`kvoadm` makes a one-time join token through the API (as you, a cluster admin), installs the signed
`kevasod` like `init` does, and starts it as a worker. It joins, and appears in:

```bash
bin/kvoadm kvo get hosts
```

It shows each host's internal and external IP; `kvoadm host list` shows them too, with the zone and status.

---

## Where silo disks live

A microVM's disks and its `local` volumes live in a volume group named `kevaso` on the worker. Kevaso
makes it once, at host setup, in one of two ways:

| You give | Kevaso does |
|---|---|
| `--storage-device /dev/DISK` | Uses that whole disk: makes the volume group on it |
| `--loop-storage SIZE` | Development only: makes the volume group on a sparse file of that size (`/var/lib/kevaso-storage.img`). Don't use it for real data |
| Neither, and a `kevaso` volume group already exists | Uses that volume group as it is |
| Neither, and none exists | Stops: `volume group "kevaso" missing: name an empty disk for it (--storage-device), create it yourself, or allow a loop file for development` |

```bash
bin/kvoadm host add root@198.51.100.20 \
    --ssh-key ~/.ssh/kevaso_ed25519 \
    --host-key SHA256:… \
    --kevasod bin/kevasod-linux-amd64 \
    --install worker \
    --name hv2 \
    --storage-device /dev/nvme1n1
```

`--storage-device` is taken by `kvoadm init --worker`, `kvoadm host enable-worker` and
`kvoadm host add`. It takes a path under `/dev/`, and not together with `--loop-storage`. The disk must
be **whole and empty**: Kevaso never writes over a disk with anything on it, and says why when it is not:

| The disk | Refused with |
|---|---|
| Doesn't exist | `/dev/nvme1n1: no such disk` |
| Has partitions or something on top of it | `… has partitions or holders (…): Kevaso takes a whole, empty disk only` |
| Is a partition or another kind of device | `… is "…", not a whole disk` |
| Is mounted | `… is mounted on /mnt` |
| Holds a file system, RAID, LVM or partition-table signature | `… isn't empty (wipefs found: …): Kevaso never writes over a disk with anything on it` |

The flag is used only when the host has no `kevaso` volume group. On a host that has one, it changes
nothing. A volume larger than the pool has left is refused ([Volumes]({{ '/silos/volumes/#what-kevaso-refuses' | relative_url }})).
Containers don't use this storage.

[Resetting a machine](#storage) never removes a volume group on a real disk unless you add
`--wipe-storage`.

---

## Placing silos

```yaml
spec:
  placement:
    region: ca-east                 # only hosts in this region
    hostSelector: { gpu: "yes" }    # only hosts with these labels
```

The scheduler also checks the runtime is installed and Ready there, that the CPU and memory fit, and
taints and tolerations, like Kubernetes. It trusts the capacity each worker reports: `status.allocatable`,
the runtimes, and the mount classes. Among the hosts that fit, it picks the one with the most free
memory from those numbers. A large step (more than double, or under half, or a mount class added
after the first report) is recorded on the host as `ReportedChange`, reason `LargeStep`, and
placement still uses the new numbers:

```text
hv1 reported allocatable memory 8Gi → 64Gi (the scheduler places silos by what a host reports)
```

Several steps are joined with `; `.

---

## Reset a machine

`kvoadm reset` takes Kevaso off a machine, so the machine can be used again without reinstalling
its operating system. It works on a system node or a worker. It only works on Debian and Ubuntu
machines (it reads the installed packages with `dpkg`).

```bash
bin/kvoadm reset root@198.51.100.20 \
    --ssh-key ~/.ssh/kevaso_ed25519 \
    --host-key SHA256:…
```

It looks at the machine first and prints what it will remove and what it will keep. Then it asks you
to type `yes`. Nothing is changed before that. `--dry-run` stops after the plan.

After you confirm, `reset` stops `kevasod` and the fence, then looks at the machine again, before it
removes anything. The plan can sit while you read it, and a silo or a volume can appear in that
time. If the second look matches what was shown, `reset` goes on and removes what the plan listed. If it
finds a silo that was not shown, or a volume whose data would be lost that was not listed, nothing
is removed. `kevasod` and the fence are started again, and the command stops:

```text
the host changed since it was shown: silo container payments/api. Nothing was removed and kevasod is started again: run kvoadm reset again to see what it would remove now
```

If the second look itself fails, the message is `looking at the host again before removing anything: … (kevasod is started again; nothing was removed)`. `--yes` skips the question and still stops the agent and looks again.

| Flag | Meaning |
|---|---|
| `[USER@]ADDRESS` | The machine, as for `kvoadm init`. The user defaults to `root`; any other user needs passwordless `sudo`. |
| `--ssh-key PATH` | The key to log in with. Without it, `reset` uses your key from the cluster folder's `local.yaml`, else `kvoadm`'s own key in that folder, and stops otherwise. Reset never logs in with a password. |
| `--host-key SHA256:…` | The machine's host key fingerprint, as in [Prepare a machine]({{ '/setup/prepare/#step-4--write-down-the-machines-fingerprint' | relative_url }}). A different key stops the command. Without it, `reset` shows the key and asks you to trust it. |
| `--ssh-port N` | The machine's SSH port (default 22). |
| `--dry-run` | Look and print the plan, then stop: it prints `Dry run: nothing was changed.` |
| `--yes`, `-y` | Don't ask for `yes`. For scripts. |
| `--force` | Go ahead although silos are on the machine. Their data is lost. Without it, `reset` stops and names the silos. |
| `--wipe-storage` | Also remove the `kevaso` volume group where it sits on a real disk, or where it was there before Kevaso, and loop-file volumes that no silo on the host is using. See below. |

The flags that set a machine up (`--kevasod`, `--runtime`, `--worker`, `--password` and the like)
are refused: `--kevasod isn't a reset flag`.

### What it removes

- `kevasod`, the fence and every `kevaso-*` unit, and anything of Kevaso's still running (`kevasod`,
  `cloud-hypervisor`, `ch-remote`, `virtiofsd`).
- Kevaso's containers and their volumes, found by the label `kevaso.com/uid`, on Docker and on podman.
- Kevaso's network: links named `kvbr*`, `kvt*` and `kevaso*`, addresses labelled `:kv`, and the
  nftables tables Kevaso added.
- Kevaso's files: `/usr/local/bin/kevasod`, `/usr/local/bin/kevaso-plugin-*`, `/usr/local/lib/kevaso`,
  `/etc/kevaso`, `/etc/kevaso-silos.nft`, `/etc/profile.d/kevaso-secrets.sh`, its systemd units,
  `/opt/kevaso`, `/var/lib/kevaso` and `/var/lib/kevaso-*`, `/var/cache/kevaso`, `/run/kevaso` and
  `/run/kevaso-*`. Anything mounted under them is unmounted first.
- The `kevaso-*` users and groups.
- From the [install record](#the-install-record): the packages Kevaso's runtimes installed, and
  the setting `net.ipv4.ip_forward` back to what it was. If Kevaso installed Docker, every
  container, volume and image goes, then Docker, `/var/lib/docker`, `/var/lib/containerd` and
  `/etc/docker`. If it installed podman, the same for podman and `/var/lib/containers`. If Docker was
  already there, it stays and its `daemon.json` is put back as it was (or removed, if there was none).
  Putting that file back restarts Docker, which stops containers that were running. Containers that
  are not Kevaso's are started again. `reset` prints `started foreign container NAME again` for each.
  One that does not start fails that step.

Nothing under Kevaso's paths is removed while something is still mounted there: `reset` unmounts
first, and if a mount stays it stops that step and says `something is still mounted under Kevaso's paths: not removing them`, so a volume mounted inside them is never reached. Files are removed without crossing into
another file system.

A step that fails does not stop the later removal steps. The install record is kept when any step
failed, so the next `kvoadm reset` still knows what to undo. When the steps are done, `reset` runs a
checklist (no Kevaso unit, process, file, mount, link, address, nftables table, container, loop
device, user or volume group left; nothing listening on 2379, 2380 or 6443) and names anything it
finds. If a step failed, the command fails, even when the checklist is clean, and prints
`Steps that failed`. When every step and the checklist pass, it prints
`→ ADDRESS is clean: every checklist item passed. kvoadm init or host add can set it up again.`

### What it never removes

It never touches `sshd` or your SSH keys, the machine's own firewall tables, packages Kevaso did not
install (a kernel the machine updated meanwhile, for instance), or data outside the paths above. It
does not talk to the cluster: the cluster still lists the host until you run `kvoadm host remove NAME`.
It changes nothing on your own computer
(`~/.kevaso` stays).

### Storage

Silo disks of the microVM runtime live in a volume group named `kevaso`.

| Where the volume group is | What `reset` does | With `--wipe-storage` |
|---|---|---|
| None, and no `/var/lib/kevaso-storage.img` | Nothing of the volume group to remove | Same |
| On `/var/lib/kevaso-storage.img`, the loop file `--loop-storage` made, and it did not exist before Kevaso, and it has volumes or its volume group can't be read, and no silo is on the host | **Refuses**. It names the volumes, or the storage file if the volume group can't be read. Look at them first (`lvs kevaso`), then reset with `--wipe-storage` | Removes the volume group with every volume on it, detaches the loop device and deletes the file. The volumes are listed, and are lost |
| On that same loop file, with no volumes, or with a silo on the host and `--force` | Removes the volume group with every volume on it, detaches the loop device and deletes the file. The volumes are listed, and are lost | Same |
| On a real disk (named with `--storage-device`), or it existed before Kevaso (the record says so) | **Keeps** the volume group and its volumes, and says so | Removes the volume group with every volume on it, then the physical volumes' LVM labels (`lvremove`, `vgremove`, `pvremove`) |

`--wipe-storage` removes only the volume group named `kevaso`, and only LVM's records of it: it does
not repartition the disk or overwrite its data, and it touches no other volume group. It removes
the loop file too. Where the table removes volumes, the silos' data on them is gone.

### The install record

Before `kvoadm init` or `kvoadm host add` installs anything, it writes `/var/lib/kevaso-install/record`
on the machine (root only). It holds the machine as it was: its installed packages, its nftables
tables, `net.ipv4.ip_forward`, whether the `kevaso` volume group already existed, and a copy of
Docker's `daemon.json` if there was one. `reset` reads it to remove what Kevaso added and nothing
else, and deletes it with the rest.

- A second install on the same machine keeps the first record.
- Kevaso installed here before the record existed, with no record: `reset` says
  `NO INSTALL RECORD` and uses fixed lists only. It deletes nftables tables named `kevaso*`, purges no
  package, leaves IP forwarding alone, and removes Docker's `daemon.json` only if it is exactly what
  Kevaso writes.

### What has run

The plan `reset` runs is the one the bare-metal test suite uses to clean its machine; the 13
one-machine tests passed and the machine came back clean. On bare metal it has also reset a named
disk, with and without `--wipe-storage`. Beside a Docker that was already installed, containers that
are not Kevaso's come back, and a failed step fails the reset and keeps the install record.
