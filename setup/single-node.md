---
title: Single system node
permalink: /setup/single-node/
---

# Single system node

With a [prepared machine]({{ '/setup/prepare/' | relative_url }}) (your Kevaso key on it, its
fingerprint written down), install the control plane. Then decide whether
that machine also runs silos: see [Choosing a setup]({{ '/setup/' | relative_url }}) for what that
means for fault tolerance and security.

## Step 1 · Install the control plane

```bash
bin/kvoadm init root@203.0.113.10 \
    --ssh-key ~/.ssh/kevaso_ed25519 \
    --host-key SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4 \
    --kevasod bin/kevasod-linux-amd64 \
    --name hv1 \
    --cluster home
```

| Flag | Meaning |
|---|---|
| `--name hv1` | The host's name in Kevaso (default: its address with dashes) |
| `--cluster home` | The cluster's name, and the name of your `kvo` context |
| `--api-server URL` | Only if *you* reach the API at another address than the machine's (a tunnel, a DNS name) |
| `--internal-ip IP` | The machine's address on its own network: the one other machines in that network use |
| `--external-ip IP` | The machine's address reachable from outside that network, if it has one |
| `--user NAME` | Your name in Kevaso (default: your login name on this computer) |

With neither `--internal-ip` nor `--external-ip`, the SSH address becomes the internal IP, if it is an
IPv4 address. If you reach the machine by a name or a tunnel, give at least one of them, or `init`
stops. Both must be IPv4 addresses, and they must differ. What they mean is on [Addresses, ports and names]({{ '/networking/model/' | relative_url }}).

**First, it looks.** Before changing anything, `init` reads the machine and shows what it found:

```
What's on this machine:
  ✓ operating system    Ubuntu 24.04.1 LTS
  ✓ CPU                 x86_64            build kevasod with GOARCH=amd64
  ✓ account             root
  ✓ systemd             yes
  ✓ openssl             yes
  ✓ KVM                 /dev/kvm          microVM silos (cloud-hypervisor, openshell) can run here
  ✓ size                4 CPUs, 7953 MiB memory, 71 GiB free   a control plane alone needs ~1 CPU and 1 GiB; silos need their own
  ! SSH password login  on                anyone can try passwords: set PasswordAuthentication no once your key works
```

| Mark | Means |
|---|---|
| `✓` | Fine |
| `!` | Works, but read the note |
| `✗` | Won't work: `init` stops there and changes nothing |

The operating system line is judged by the list of [supported systems]({{ '/setup/prepare/#supported-operating-systems' | relative_url }}):
an unlisted system gets a `!`, not a stop. To only look, add `--dry-run`: it stops after this table.

> **Password logins on?** Now that your key works, you can turn them off: `kvoadm init` asks you
> (or pass `--disable-password-login`). Or do it yourself: on the machine, put
> `PasswordAuthentication no` in `/etc/ssh/sshd_config.d/00-local.conf` and run
> `systemctl reload ssh`, keeping your current session open until a new key login works.

In a terminal, it first asks two questions, both **No** unless you say otherwise:

```
Turn off SSH password logins on this machine?   Keys only from then on.
Change root's password to a random one?        Nobody will know it, you included.
```

Kevaso doesn't need either; they're extra hardening for a machine that's now Kevaso's. In a script,
nothing is asked: pass `--disable-password-login` and `--change-root-password` for the ones you
want, or `--no-extra-hardening` to skip the questions.

What it does, in order (each line is printed as it happens):

1. **Connects** with your key, checking the host key against `--host-key`.
2. **Checks** the CPU matches the `kevasod` you built.
3. **Records the machine as it is**, before installing anything: its packages, nftables tables, IP
   forwarding, whether a `kevaso` volume group exists, Docker's `daemon.json`. It is kept in
   `/var/lib/kevaso-install/record` (root only), and [`kvoadm reset`]({{ '/workers/#reset-a-machine' | relative_url }})
   reads it. A second install keeps the first record.
4. **Installs `kevasod`**, signed: `kvoadm` signs the binary with a key of its own
   (`~/.kevaso/signing.key`, made on first use), pins the public half on the machine
   (`/etc/kevaso/release.pub`) and the machine verifies the signature before installing to
   `/usr/local/bin/kevasod`. A binary changed on the way doesn't get installed.
5. **Creates the cluster** on the machine and **starts the control plane** as `kevasod.service`.
   Creating is its own step (`→ cluster NAME created on this host`): it leaves a record that this
   machine belongs to cluster NAME. The first start then makes the cluster's private CA, its store,
   and the master key for secrets, all in `/var/lib/kevaso` (root only). The service never creates a
   cluster by itself: [Starting a cluster, and a node that finds its data gone]({{ '/operations/backup/#starting-a-cluster-and-a-node-that-finds-its-data-gone' | relative_url }}).
6. **Gets you an admin certificate**: the private key is made *on your computer*; only a signing
   request goes to the machine, which signs it through its root-only admin socket.
7. **Writes your `kvo` context** to `~/.kevaso/config.yaml`: the API address, the cluster CA (so
   `kvo` trusts only *this* cluster) and your certificate.
8. **Waits for the host to be Ready**, then prints the **recovery key** (next step).

Now try `kvo`:

```bash
bin/kvo get hosts
```

```
NAME   INTERNAL IP    EXTERNAL IP   REGION   COMPONENTS      RUNTIMES   AGE
hv1    203.0.113.10   -             -        control-plane   -          1m
```

### Running init again

A second `kvoadm init` on a machine that already has `kevasod.service` keeps what the unit already
runs: the worker, its runtimes, its storage (`--storage-device` or `--loop-storage`), the store
(`sqlite` or `etcd`), the host name, the internal and external addresses, and the etcd peer listen
and join flags. A flag you leave out stays as the unit has it.

A value you pass that differs is refused, and nothing is written:

```text
kvoadm: kevasod.service on this machine has --name hv1 (the installed unit) and --name hv9 (this init): pass the same value, or --replace-unit to write the unit from this run only
```

A run that would drop a worker, a runtime, the storage mode, or an etcd store is refused the same
way:

```text
kvoadm: kevasod.service on this machine has --worker, --runtime cloud-hypervisor, and this init would drop them: repeat those flags, or pass --replace-unit to write the unit from this run only
```

`--replace-unit` writes the unit from this run's flags only. The same keep applies when
`kvoadm init --restore` finds a unit already installed.

---

## Step 2 · Save the recovery key

The last thing `init` prints is this:

```
Recovery key for secrets (the spare key to them; shown only now, keep it offline):

    KVR1-3NQF-…
```

**Save it now.** It's shown this once, and nothing can show it again: the machine keeps only a
lock that it opens, not the key itself.

It's the key to your backups. Without it:

- a backup can't be restored, since backups are sealed to it;
- if the machine is lost, so are your secrets, even with a backup in hand.

Where to keep it:

| Good | Not good |
|---|---|
| A password manager entry of its own (not the one on your laptop alone) | A file next to your backups |
| Printed, in a safe, with your other recovery codes | Chat, email, a ticket, the repository |
| Two places, held by two people, for a team | The control-plane machine itself |

Someone with the recovery key *and* a backup can read every secret in it. Keep the two apart. If
the key leaks, replace it:
[`kvoadm kms rotate … --new-recovery-key`]({{ '/secrets/' | relative_url }}#the-master-key-and-the-recovery-key).

Then take your first backup: [Backup and restore]({{ '/operations/backup/' | relative_url }}).

---

## Step 3 · Let the machine run silos (optional)

A control plane alone runs no silos. That's the [recommended setup]({{ '/setup/' | relative_url }}#dedicated-system-node--workers):
[add workers]({{ '/workers/' | relative_url }}) next. For a one-machine cluster, a
[combined node]({{ '/setup/' | relative_url }}#combined-node-control-plane-and-silos-on-one-machine),
let it be a worker too:

```bash
bin/kvoadm host enable-worker root@203.0.113.10 \
    --ssh-key ~/.ssh/kevaso_ed25519 \
    --host-key SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4 \
    --runtime cloud-hypervisor
```

`--runtime` picks what kind of silos it runs (repeat it for several; see [Runtimes]({{ '/silos/runtimes/' | relative_url }})):

| Runtime | Needs |
|---|---|
| `cloud-hypervisor` | KVM, and a volume group named `kevaso` for silo disks: name an empty disk with `--storage-device /dev/DISK` at setup, or create it yourself ([Where silo disks live]({{ '/workers/#where-silo-disks-live' | relative_url }})). `--loop-storage 40G` uses a file instead, for trying things out |
| `openshell`, `openshell-container` | KVM for the first; Docker for the second (installed for you) |
| `docker`, `podman` | Nothing special (installed for you) |

The worker installs its runtimes in the background (packages, Cloud Hypervisor, the Ubuntu image).
Watch it:

```bash
bin/kvo watch hosts
```

Wait for `cloud-hypervisor=Ready`, then make [your first silo]({{ '/silos/first-silo/' | relative_url }}).

> **In one command.** `kvoadm init … --worker --runtime cloud-hypervisor` does steps 1 and 3
> together, and produces exactly the same setup. The steps are separate here so you can see each
> one.

---
