---
title: Quick start
permalink: /quickstart/
---

# Quick start

A whole cluster on your Mac in about 15 minutes: a Linux VM that is both the system node and a
worker, and a microVM silo inside it. The [Running locally]({{ '/setup/local/' | relative_url }})
page explains every step in more depth.

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
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -o bin/kevasod-linux-arm64 ./cmd/kevasod
```

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
throwaway VM), asks whether to harden SSH (answer No here), then installs. `--api-server` is where
*you* reach the API: Lima forwards the VM's port 6443 to your Mac. At the end it prints a recovery
key; for a throwaway VM you can let it go.

```bash
bin/kvo get hosts
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

The worker now installs Cloud Hypervisor and an Ubuntu image. Watch it:

```bash
bin/kvo watch hosts
```

When you see `cloud-hypervisor=Ready`, press Ctrl-C.

## 6 · Your first silo

Save this as `hello.yaml`:

```yaml
apiVersion: kevaso/v1
kind: Namespace
metadata: { name: demo }
---
apiVersion: kevaso/v1
kind: Silo
metadata: { name: hello, namespace: demo }
spec:
  runtime: cloud-hypervisor
  image: ubuntu-24.04
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

```bash
rm -rf ~/.kevaso
```

---

**Next:** [Concepts]({{ '/concepts/' | relative_url }}) for the words, or
[Choosing a setup]({{ '/setup/' | relative_url }}) for a real cluster.
