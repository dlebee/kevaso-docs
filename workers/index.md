---
title: Adding workers
permalink: /workers/
---

# Adding workers

One control plane, any number of workers: a server in a data center, a VPS, a box at home behind
your router. They don't need to be on the same network.

---

## How workers connect

- A new worker **joins with a one-time token** and gets a certificate for exactly its own name and
  role. It can't act as another host, and it can't touch silos placed elsewhere.
- It **dials out** to the control plane and keeps that connection open; `kvo exec` and everything
  else come back through it. Nothing needs to listen on the worker.
- Hosts are on a **WireGuard mesh** run by the control plane. Each host gets its own silo subnet
  (`10.64.N.0/24`), so silos on different hosts reach each other by address and name.
- A worker **behind NAT** reaches the others through a relay (a host that can be reached directly),
  end to end encrypted inside it: the relay only ever sees encrypted UDP.

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

`kvoadm` makes a one-time join token through the API (as you, a cluster admin), installs the signed
`kevasod` like `init` does, and starts it as a worker. It joins, and appears in:

```bash
bin/kvo get hosts
```

It shows each host's internal and external IP; `kvoadm host list` shows them too, with the zone and status.

---

## Placing silos

```yaml
spec:
  placement:
    region: ca-east                 # only hosts in this region
    hostSelector: { gpu: "yes" }    # only hosts with these labels
```

The scheduler also checks the runtime is installed and Ready there, that the CPU and memory fit, and
taints and tolerations, like Kubernetes.

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
to type `yes`. Nothing is changed before that.

| Flag | Meaning |
|---|---|
| `[USER@]ADDRESS` | The machine, as for `kvoadm init`. The user defaults to `root`; any other user needs passwordless `sudo`. |
| `--ssh-key PATH` | The key to log in with. Without it, `reset` uses `~/.kevaso/ssh/id_ed25519` if that file exists, and stops otherwise. Reset never logs in with a password. |
| `--host-key SHA256:…` | The machine's host key fingerprint, as in [Prepare a machine]({{ '/setup/prepare/#step-4--write-down-the-machines-fingerprint' | relative_url }}). A different key stops the command. Without it, `reset` shows the key and asks you to trust it. |
| `--ssh-port N` | The machine's SSH port (default 22). |
| `--dry-run` | Look and print the plan, then stop: it prints `Dry run: nothing was changed.` |
| `--yes`, `-y` | Don't ask for `yes`. For scripts. |
| `--force` | Go ahead although silos are on the machine. Their data is lost. Without it, `reset` stops and names the silos. |
| `--wipe-storage` | Also remove the `kevaso` volume group where it sits on a real disk, or where it was there before Kevaso. See below. |

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

A step that fails does not stop the others. When all steps are done, `reset` runs a checklist
(no Kevaso unit, process, file, mount, link, address, nftables table, container, loop device, user or
volume group left; nothing listening on 2379, 2380 or 6443) and names anything it finds. Then it ends
with `<ADDRESS> is clean`. If something is left, it prints what and exits with an error.

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
| None | Removes only a leftover `/var/lib/kevaso-storage.img` | Same |
| On `/var/lib/kevaso-storage.img`, the loop file `--loop-storage` made, and it did not exist before Kevaso | Removes the volume group with every volume on it, detaches the loop device and deletes the file. The volumes are listed, and are lost. | Same |
| On a real disk, or it existed before Kevaso (the record says so) | **Keeps** the volume group and its volumes, and says so | Removes the volume group with every volume on it, then the physical volumes' LVM labels (`lvremove`, `vgremove`, `pvremove`) |

`--wipe-storage` removes only the volume group named `kevaso`, and only LVM's records of it: it does
not repartition the disk or overwrite its data, and it touches no other volume group. It removes
the loop file too. The silos' data on those volumes is gone either way.

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
one-machine tests passed and the machine came back clean. The command itself has been tested only
against a stand-in for the machine. It has not been run on a machine holding a real cluster.
