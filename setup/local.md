---
title: Running locally
permalink: /setup/local/
---

# Running locally

Two ways to run Kevaso on your own computer, depending on what you want to try.

| | A Linux VM with everything | `kevasod dev` |
|---|---|---|
| **What runs** | A real control plane and worker, in a Lima VM | Only the API, on your computer |
| **Silos** | Yes: real microVMs (with nested virtualization) or containers | No: objects are stored, nothing runs |
| **Setup** | About 15 minutes | Seconds |
| **Good for** | Learning Kevaso end to end | Trying `kvo`, writing YAML, developing the API |

Neither is for anything you'd mind losing: see [Choosing a setup]({{ '/setup/' | relative_url }}#running-locally).

---

## A Linux VM with everything

The [quick start]({{ '/quickstart/' | relative_url }}) has every command in order. This section
explains what each one is for.

### The VM

On a Mac, [Lima](https://lima-vm.io) runs Linux VMs with Apple's own hypervisor. With **nested
virtualization** (Apple M3 or later, macOS 15 or later), the VM has KVM, so it can run microVM
silos itself.

```bash
limactl create \
    --name=kevaso-local \
    --vm-type=vz \
    --nested-virt \
    --cpus=4 \
    --memory=6 \
    --disk=60 \
    --mount-none template:ubuntu-24.04
```

| Option | Why |
|---|---|
| `--vm-type=vz` | Apple's hypervisor (Virtualization.framework) |
| `--nested-virt` | KVM inside the VM, for microVM silos |
| `--mount-none` | Don't share your home folder with the VM: Kevaso treats it as its own machine |
| `--cpus=4 --memory=6 --disk=60` | Room for the control plane and a couple of silos |
| `template:ubuntu-24.04` | The Linux Kevaso is tested on |

### The way in

This is [Prepare a machine]({{ '/setup/prepare/' | relative_url }}), with Lima doing the first steps
for you.

**The SSH key.** Lima made one when you first used it, `~/.lima/_config/user`, and put its public
half in every VM it creates. For a throwaway VM that's fine. For a real machine you'd
[make one for Kevaso]({{ '/setup/prepare/' | relative_url }}#step-2--make-an-ssh-key-for-kevaso).

**The address.** Lima forwards the VM's SSH to a port on your Mac:

```bash
limactl list kevaso-local
```

```
NAME           STATUS     SSH                VMTYPE    ARCH       CPUS    MEMORY    DISK
kevaso-local   Running    127.0.0.1:60022    vz        aarch64    4       6GiB      60GiB
```

So you log in as your Mac user, at `127.0.0.1`, port `60022` (yours will differ).

**The fingerprint.** `limactl shell` reaches the VM through Lima itself, not the network, so it's a
trustworthy place to read the VM's identity:

```bash
limactl shell kevaso-local ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

```
256 SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4 root@lima-kevaso-local (ED25519)
```

The `SHA256:…` part is what you give `kvoadm` as `--host-key`.

### Check, install, add the worker

These are the same commands as for any machine, with Lima's port and key:

```bash
bin/kvoadm check you@127.0.0.1 \
    --ssh-port 60022 \
    --ssh-key ~/.lima/_config/user \
    --host-key SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4
```

```bash
bin/kvoadm init you@127.0.0.1 \
    --ssh-port 60022 \
    --ssh-key ~/.lima/_config/user \
    --host-key SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4 \
    --kevasod bin/kevasod-linux-arm64 \
    --name local \
    --cluster local \
    --api-server https://127.0.0.1:6443
```

`--api-server https://127.0.0.1:6443`: Lima forwards the VM's API port to your Mac's loopback, so
that's where `kvo` reaches it.

```bash
bin/kvoadm host enable-worker you@127.0.0.1 \
    --ssh-port 60022 \
    --ssh-key ~/.lima/_config/user \
    --host-key SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4 \
    --runtime cloud-hypervisor \
    --loop-storage 40G
```

`--loop-storage 40G`: silo disks normally live in a volume group named `kevaso`. In a VM, a 40 GB
sparse file stands in for it.

### Clean up

```bash
limactl delete --force kevaso-local
```

```bash
rm -rf ~/.kevaso
```

---

## `kevasod dev`: just the API

No machines and no silos: the API alone, on your loopback, with no authentication. Objects are
stored (in `~/.kevaso/dev.db`) and checked like on a real cluster, so it's the fastest way to learn
the YAML.

Start it, and leave it running:

```bash
go run ./cmd/kevasod dev
```

In another terminal, point `kvo` at it:

```bash
export KVO_SERVER=http://127.0.0.1:8080
```

Save this as `demo.yaml`:

```yaml
apiVersion: kevaso/v1
kind: Namespace
metadata: { name: demo }
```

```bash
bin/kvo apply -f demo.yaml
```

```bash
bin/kvo get ns
```

It only listens on loopback, on purpose: there's no sign-in in dev mode.
