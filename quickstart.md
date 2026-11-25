---
title: Quick start
permalink: /quickstart/
audience: both
---

# Quick start

A whole cluster on your Mac in about 15 minutes: a Linux VM that is both the system node and a
worker, and a microVM silo inside it. The [Running locally]({{ '/admin/setup/local/' | relative_url }})
page explains every step in more depth.

> **This quick start is for a Mac:** it runs the cluster in a Lima VM on Apple's own hypervisor.
> **On Linux or Windows,** use any Ubuntu 24.04 or Debian 12 machine you can reach over SSH (a VM, a
> spare computer, a cloud machine): [prepare it]({{ '/admin/setup/prepare/' | relative_url }}), then install a
> [single system node]({{ '/admin/setup/single-node/' | relative_url }}) on it. The tools for your
> computer are in [Get the tools]({{ '/admin/setup/prepare/' | relative_url }}#step-1--get-the-tools).

**You need:** an Apple Silicon Mac (M3 or later, macOS 15 or later, for nested virtualization),
[Homebrew](https://brew.sh), Go 1.27, and a checkout of
[dlebee/kevaso](https://github.com/dlebee/kevaso). Run the commands from that checkout.

---

## 1 · Build the tools

```bash
go build -o bin/kvo ./cmd/kvo
```

```bash
go build -o bin/kvoadm ./cmd/kvoadm
```

`kevasod` runs inside the Linux VM, so build it for Linux on ARM:

```bash
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -ldflags "-X kevaso.com/core/internal/version.Version=vX.Y.Z" -o bin/kevasod-linux-arm64 ./cmd/kevasod
```

Or take all three from a [release]({{ '/admin/setup/prepare/' | relative_url }}#step-1--get-the-tools)
(`kvo` and `kvoadm` for `darwin-arm64`, `kevasod` for `linux-arm64`), and skip Go.

## 2 · Make a Linux VM

Install [Lima](https://lima-vm.io), which runs Linux VMs on a Mac:

```bash
brew install lima
```

Create the VM and start it:

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

If Lima asks how to proceed, choose **Proceed with the current configuration**.

```bash
limactl start kevaso-local
```

## 3 · Find the way in

Lima already made an SSH key and put it in the VM. Ask Lima where the VM's SSH is:

```bash
limactl list kevaso-local
```

```
NAME           STATUS     SSH                VMTYPE    ARCH       CPUS    MEMORY    DISK
kevaso-local   Running    127.0.0.1:60022    vz        aarch64    4       6GiB      60GiB
```

Note the port (`60022` here; yours will differ). The user is your Mac login name, and the key is
`~/.lima/_config/user`.

Now the VM's fingerprint, its identity, read through Lima rather than over the network:

```bash
limactl shell kevaso-local ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

```
256 SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4 root@lima-kevaso-local (ED25519)
```

Copy the `SHA256:…` part. In the next commands, replace `you` with your Mac login name, `60022` with
your port and the fingerprint with yours.

## 4 · Install the control plane

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

It first shows what it found on the VM (all `✓`; a `!` for SSH password login is fine for a
throwaway VM), asks two hardening questions (answer No to both here), then installs. `--api-server` is where
*you* reach the API: Lima forwards the VM's port 6443 to your Mac.

> **Save the recovery key.** The last thing `init` prints is a line starting `KVR1-`: the recovery
> key. It's shown only this once. Put it in your password manager now, even for this practice VM, so
> it's a habit. On a real cluster it's the only thing that opens your backups: lose it and a lost
> control plane can't be restored, nor its secrets
> ([more]({{ '/admin/setup/single-node/' | relative_url }}#step-2--save-the-recovery-key)).

```bash
bin/kvoadm kvo get hosts
```

## 5 · Let it run silos

```bash
bin/kvoadm host enable-worker you@127.0.0.1 \
    --ssh-port 60022 \
    --ssh-key ~/.lima/_config/user \
    --host-key SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4 \
    --runtime cloud-hypervisor \
    --loop-storage 40G
```

The worker now installs Cloud Hypervisor and an Ubuntu image. Watch it (hosts are the admin's, so
through `kvoadm kvo`):

```bash
bin/kvoadm kvo watch hosts
```

When you see `cloud-hypervisor=Ready`, press Ctrl-C.

## 6 · Your first silo

Silos live in a namespace, and making one is the admin's. Save this as `demo.yaml`:

```yaml
apiVersion: kevaso/v1
kind: Namespace
metadata: { name: demo }
```

```bash
bin/kvoadm kvo apply -f demo.yaml
```

Your own `kvo` makes the silo. Save this as `hello.yaml`:

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: hello, namespace: demo }
spec:
  runtime: cloud-hypervisor
  vm: { image: ubuntu-24.04 }
  resources: { cpu: 1, memory: 1Gi }
  disk: { size: 10Gi }
  init:
    steps:
      - name: hello
        run: echo "hello from $(hostname)" > /root/hello
```

```bash
bin/kvo apply -f hello.yaml
```

```bash
bin/kvo watch silos -n demo
```

When it says `phase=Running`, press Ctrl-C and look inside:

```bash
bin/kvo exec silo hello -n demo -- cat /root/hello
```

```
hello from hello
```

A microVM, inside a VM, on your Mac.

## 7 · Clean up

```bash
limactl delete --force kevaso-local
```

Then forget the cluster on your Mac: its folder holds the admin certificate and keys.

```bash
rm -rf ~/.kevaso/clusters/local
```

Not all of `~/.kevaso`: other clusters' folders live there too. `kvo` keeps a `local` context in
`~/.kevaso/config.yaml` that points at nothing now; `bin/kvo ctx` lists the contexts.

---

**Next:** [Concepts]({{ '/concepts/' | relative_url }}) for the words. Then
[Using Kevaso]({{ '/use/' | relative_url }}) for what you do with `kvo` (steps 6 on), or
[Administration]({{ '/admin/' | relative_url }}) and [Choosing a setup]({{ '/admin/setup/' | relative_url }})
for a real cluster (steps 1 to 5).
