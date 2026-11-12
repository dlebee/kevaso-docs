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
| **Setup** | ~10 minutes | Seconds |
| **Good for** | Learning Kevaso end to end, the [quick start]({{ '/quickstart/' | relative_url }}) | Trying `kvo`, writing YAML, developing the API |

Neither is for anything you'd mind losing: see [Choosing a setup]({{ '/setup/' | relative_url }}#running-locally).

---

## A Linux VM with everything

On a Mac, [Lima](https://lima-vm.io) runs Linux VMs with Apple's own hypervisor. With **nested
virtualization** (Apple M3 or later, macOS 15 or later), the VM has KVM, so it can run microVM
silos itself.

### 1 · Install Lima and build the tools

```bash
brew install lima
```

```bash
go build -o bin/kvo ./cmd/kvo && go build -o bin/kvoadm ./cmd/kvoadm && GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -o bin/kevasod-linux-arm64 ./cmd/kevasod
```

(`arm64`: an Apple Silicon Mac's VM is ARM.)

### 2 · Create the VM

```bash
limactl create --name=kevaso-local --vm-type=vz --nested-virt --cpus=4 --memory=6 --disk=60 --mount-none --tty=false template:ubuntu-24.04 && limactl start kevaso-local
```

| Flag | Why |
|---|---|
| `--vm-type=vz --nested-virt` | Apple's hypervisor, with KVM inside the VM |
| `--mount-none` | Don't share your home folder with the VM: Kevaso treats it as its own machine |
| `--cpus=4 --memory=6 --disk=60` | Room for the control plane and a couple of silos |

### 3 · The way in: Lima's key and the VM's fingerprint

This is the [Prepare a machine]({{ '/setup/prepare/' | relative_url }}) steps, with Lima doing
the first ones for you:

- **The SSH key:** Lima made one when you installed it (`~/.lima/_config/user`) and put it in the
  VM. For a throwaway VM that's fine; for a real machine you'd [make one for
  Kevaso]({{ '/setup/prepare/' | relative_url }}#step-2--make-an-ssh-key-for-kevaso).
- **The address:** Lima forwards the VM's SSH to a port on `127.0.0.1`. Read it from Lima's
  config:

```bash
eval "$(awk '/^ *(Hostname|Port|User|IdentityFile) /{gsub(/"/,"",$2); print "SSH_"toupper($1)"="$2}' ~/.lima/kevaso-local/ssh.config)" && echo "$SSH_USER@$SSH_HOSTNAME port $SSH_PORT key $SSH_IDENTITYFILE"
```

- **The fingerprint:** `limactl shell` is Lima's own channel to the VM, not the network, so it's a
  trustworthy place to read it:

```bash
PIN=$(limactl shell kevaso-local ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub | awk '{print $2}') && echo "$PIN"
```

### 4 · Check, install, add the worker

```bash
bin/kvoadm check "$SSH_USER@$SSH_HOSTNAME" --ssh-port "$SSH_PORT" --ssh-key "$SSH_IDENTITYFILE" --host-key "$PIN"
```

```bash
bin/kvoadm init "$SSH_USER@$SSH_HOSTNAME" --ssh-port "$SSH_PORT" --ssh-key "$SSH_IDENTITYFILE" --host-key "$PIN" --kevasod bin/kevasod-linux-arm64 --name local --cluster local --api-server https://127.0.0.1:6443
```

`--api-server https://127.0.0.1:6443`: Lima forwards the VM's API port to your Mac's loopback, so
that's where `kvo` reaches it.

```bash
bin/kvoadm host enable-worker "$SSH_USER@$SSH_HOSTNAME" --ssh-port "$SSH_PORT" --ssh-key "$SSH_IDENTITYFILE" --host-key "$PIN" --runtime cloud-hypervisor --loop-storage 40G
```

`--loop-storage 40G`: silo disks normally live in a volume group named `kevaso`; in a VM, a 40 GB
sparse file stands in for it.

```bash
bin/kvo watch hosts
```

Wait for `cloud-hypervisor=Ready` (a few minutes: packages, Cloud Hypervisor, the Ubuntu image).
Then make [your first silo]({{ '/silos/first-silo/' | relative_url }}).

### 5 · Clean up

```bash
limactl delete -f kevaso-local && rm -rf ~/.kevaso
```

---

## `kevasod dev`: just the API

No machines, no silos: the API alone, on your loopback, with no authentication. Objects are stored
(in `~/.kevaso/dev.db`) and checked like on a real cluster, so it's the fastest way to learn the
YAML.

```bash
go run ./cmd/kevasod dev
```

In another terminal:

```bash
export KVO_SERVER=http://127.0.0.1:8080
```

```bash
printf 'apiVersion: kevaso/v1\nkind: Namespace\nmetadata: { name: demo }\n' | bin/kvo apply -f - && bin/kvo get ns
```

It only listens on loopback, on purpose: there's no sign-in in dev mode.
