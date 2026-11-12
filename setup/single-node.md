---
title: Single system node
permalink: /setup/single-node/
---

# Single system node

With a [prepared machine]({{ '/setup/prepare/' | relative_url }}) (your Kevaso key on it, its
fingerprint written down, `kvoadm check` all green), install the control plane. Then decide whether
that machine also runs silos: see [Choosing a setup]({{ '/setup/' | relative_url }}) for what that
means for fault tolerance and security.

## Step 6 · Install the control plane

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
| `--address IP` | The address *other hosts* should use for this one, if not the SSH address |
| `--user NAME` | Your name in Kevaso (default: your login name on this computer) |

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
3. **Installs `kevasod`**, signed: `kvoadm` signs the binary with a key of its own
   (`~/.kevaso/signing.key`, made on first use), pins the public half on the machine
   (`/etc/kevaso/release.pub`) and the machine verifies the signature before installing to
   `/usr/local/bin/kevasod`. A binary changed on the way doesn't get installed.
4. **Starts the control plane** as `kevasod.service`. On first start it makes the cluster's
   private CA, its store, and the master key for secrets, all in `/var/lib/kevaso` (root only).
5. **Gets you an admin certificate**: the private key is made *on your computer*; only a signing
   request goes to the machine, which signs it through its root-only admin socket.
6. **Writes your `kvo` context** to `~/.kevaso/config.yaml`: the API address, the cluster CA (so
   `kvo` trusts only *this* cluster) and your certificate.
7. **Waits for the host to be Ready**, then prints the **recovery key**:

```
Recovery key for secrets (the spare key to them; shown only now, keep it offline):

    KVR1-3NQF-…
```

Put the recovery key in your password manager or on paper, **now**. It's shown once and never
again. [Secrets]({{ '/secrets/' | relative_url }}#the-master-key-and-the-recovery-key) explains what it's for.

Now try `kvo`:

```bash
bin/kvo get hosts
```

```
NAME   ADDRESS        REGION   COMPONENTS      RUNTIMES   AGE
hv1    203.0.113.10   -        control-plane   -          1m
```

---

## Step 7 · Let the machine run silos (optional)

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
| `cloud-hypervisor` | KVM, and a volume group named `kevaso` for silo disks (or `--loop-storage 40G` to use a file, for trying things out) |
| `openshell`, `openshell-container` | KVM for the first; Docker for the second (installed for you) |
| `docker`, `podman` | Nothing special (installed for you) |

The worker installs its runtimes in the background (packages, Cloud Hypervisor, the Ubuntu image).
Watch it:

```bash
bin/kvo watch hosts
```

Wait for `cloud-hypervisor=Ready`, then make [your first silo]({{ '/silos/first-silo/' | relative_url }}).

> **In one command.** `kvoadm init … --worker --runtime cloud-hypervisor` does steps 6 and 7
> together, and produces exactly the same setup. The steps are separate here so you can see each
> one.

---
