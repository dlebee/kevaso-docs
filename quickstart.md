---
title: Quick start
permalink: /quickstart/
---

# Quick start

A whole cluster on your Mac in about 15 minutes: a Linux VM that's both the system node and a
worker, and a microVM silo inside it. Every command is one line; the
[Running locally]({{ '/setup/local/' | relative_url }}) page explains each one.

**You need:** an Apple Silicon Mac (M3 or later, macOS 15 or later, for nested virtualization), Go
1.27, and a checkout of [dlebee/kevaso](https://github.com/dlebee/kevaso). Run everything from
that checkout.

---

## 1 · Tools and a VM

```bash
brew install lima
```

```bash
go build -o bin/kvo ./cmd/kvo && go build -o bin/kvoadm ./cmd/kvoadm && GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -o bin/kevasod-linux-arm64 ./cmd/kevasod
```

```bash
limactl create --name=kevaso-local --vm-type=vz --nested-virt --cpus=4 --memory=6 --disk=60 --mount-none --tty=false template:ubuntu-24.04 && limactl start kevaso-local
```

## 2 · The way in

The VM's SSH address and key (Lima made them), and its host-key fingerprint, read through Lima's own
channel:

```bash
eval "$(awk '/^ *(Hostname|Port|User|IdentityFile) /{gsub(/"/,"",$2); print "SSH_"toupper($1)"="$2}' ~/.lima/kevaso-local/ssh.config)" && PIN=$(limactl shell kevaso-local ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub | awk '{print $2}') && echo "$PIN"
```

## 3 · Look, then install

```bash
bin/kvoadm check "$SSH_USER@$SSH_HOSTNAME" --ssh-port "$SSH_PORT" --ssh-key "$SSH_IDENTITYFILE" --host-key "$PIN"
```

All `✓` (a `!` for password login is fine here). Then the control plane:

```bash
bin/kvoadm init "$SSH_USER@$SSH_HOSTNAME" --ssh-port "$SSH_PORT" --ssh-key "$SSH_IDENTITYFILE" --host-key "$PIN" --kevasod bin/kevasod-linux-arm64 --name local --cluster local --api-server https://127.0.0.1:6443
```

It prints a recovery key at the end; for a throwaway VM you can let it go. And the worker, on the
same VM:

```bash
bin/kvoadm host enable-worker "$SSH_USER@$SSH_HOSTNAME" --ssh-port "$SSH_PORT" --ssh-key "$SSH_IDENTITYFILE" --host-key "$PIN" --runtime cloud-hypervisor --loop-storage 40G
```

```bash
bin/kvo watch hosts
```

Wait for `cloud-hypervisor=Ready`, then Ctrl-C.

## 4 · A silo

```bash
printf 'apiVersion: kevaso/v1\nkind: Namespace\nmetadata: { name: demo }\n---\napiVersion: kevaso/v1\nkind: Silo\nmetadata: { name: hello, namespace: demo }\nspec:\n  runtime: cloud-hypervisor\n  image: ubuntu-24.04\n  resources: { cpu: 1, memory: 1Gi }\n  disk: { size: 10Gi }\n  init:\n    steps:\n      - { name: hello, run: "echo hello from $(hostname) > /root/hello" }\n' | bin/kvo apply -f -
```

```bash
bin/kvo watch silos -n demo
```

Wait for `phase=Running`, then:

```bash
bin/kvo exec silo hello -n demo -- cat /root/hello
```

`hello from hello`: a microVM, inside a VM, on your Mac.

## 5 · Clean up

```bash
limactl delete -f kevaso-local && rm -rf ~/.kevaso
```

---

**Next:** [Concepts]({{ '/concepts/' | relative_url }}) for the words, or
[Choosing a setup]({{ '/setup/' | relative_url }}) for a real cluster.
