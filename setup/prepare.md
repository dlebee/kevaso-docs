---
title: Prepare a machine
permalink: /setup/prepare/
---

# Prepare a machine

Before Kevaso changes anything on a machine, you set up the way in, yourself, in five small steps:
build the tools, make an SSH key for Kevaso, put it on the machine, write down the machine's
identity, and look at what Kevaso will find there. You do this once for every machine: the system
node and each worker.

| Step | You do | Why |
|---|---|---|
| [1](#step-1--build-the-tools) | Build `kvo`, `kvoadm` and `kevasod` | No releases yet |
| [2](#step-2--make-an-ssh-key-for-kevaso) | Make an SSH key for Kevaso | So `kvoadm` never needs a password |
| [3](#step-3--put-the-key-on-the-machine) | Put its public half on the machine | So the machine lets that key in |
| [4](#step-4--write-down-the-machines-fingerprint) | Write down the machine's host-key fingerprint | So you know you're talking to *your* machine |
| [5](#step-5--check-the-machine) | `kvoadm check` | See what Kevaso finds before it changes anything |

Then: [install the control plane]({{ '/setup/single-node/' | relative_url }}) on it, or
[add it as a worker]({{ '/workers/' | relative_url }}).

---

## Step 1 · Build the tools

From a checkout of [dlebee/kevaso](https://github.com/dlebee/kevaso), on your computer:

```bash
go build -o bin/kvo ./cmd/kvo
```

```bash
go build -o bin/kvoadm ./cmd/kvoadm
```

`kevasod` runs on the machine, so build it for Linux and for the machine's CPU (`amd64` for x86_64,
`arm64` for ARM; step 5 tells you which):

```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o bin/kevasod-linux-amd64 ./cmd/kevasod
```

> **Why `CGO_ENABLED=0`?** It makes a static binary that runs on any Linux, with nothing to install
> next to it. The same file also becomes `kvo-guest` inside your VMs later.

---

## Step 2 · Make an SSH key for Kevaso

`kvoadm` logs in to your machines over SSH to install `kevasod`. Give it a key of its own, separate
from your everyday key:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/kevaso_ed25519 -C "kevaso admin"
```

It asks for a **passphrase**: set one. You get two files:

| File | What it is | Who sees it |
|---|---|---|
| `~/.ssh/kevaso_ed25519` | The **private** key | Only you. Never copy it anywhere. |
| `~/.ssh/kevaso_ed25519.pub` | The **public** key | Every machine Kevaso manages |

> **Why a key, not a password?** A password can be guessed or phished, and it travels to the server
> each time. A key never leaves your computer: the machine only checks a signature made with it.
> **Why ed25519?** Short, fast, and what every current OpenSSH accepts.
> **Why its own key?** You can revoke it (remove one line on each machine) without touching your
> personal access, and you can tell Kevaso's logins apart in the machine's logs.
> **Why a passphrase?** If your laptop is stolen, the key file alone is useless. `kvoadm` takes the
> key from your `ssh-agent` if it's loaded there (`ssh-add ~/.ssh/kevaso_ed25519`), or asks for the
> passphrase once.

---

## Step 3 · Put the key on the machine

The machine has to accept the public key for the account `kvoadm` will use (`root`, or a user with
`sudo`, like `ubuntu` on most cloud images). Pick the way that matches how you got the machine:

**A · A cloud provider or VPS** — most let you paste a public key when you create the machine. Paste
the contents of:

```bash
cat ~/.ssh/kevaso_ed25519.pub
```

**B · A machine you can already log in to with a password** — copy the key over with OpenSSH:

```bash
ssh-copy-id -i ~/.ssh/kevaso_ed25519.pub root@203.0.113.10
```

This is the one time you type the password. Keep reading before you answer `yes` to the question it
asks: that's step 4.

Then check that the key works on its own:

```bash
ssh -i ~/.ssh/kevaso_ed25519 root@203.0.113.10
```

You should get a shell without being asked for the machine's password (only for your key's
passphrase, if `ssh-agent` doesn't have it yet). Type `exit` to leave.

> **Prefer not to do this by hand?** `kvoadm init --password` logs in with the password once,
> installs a key of its own, turns password logins off and changes the root password. It's the
> same result, done for you, and it still asks you to confirm the machine first.

---

## Step 4 · Write down the machine's fingerprint

The first time you connect to a machine, SSH shows its **host key fingerprint**, something like
`SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4`. This is the machine's identity. If someone
sits between you and the machine, the fingerprint won't match, and that's the only moment you can
notice.

So get the real fingerprint **from somewhere other than this network connection**:

| Where the machine is | Where to read the fingerprint |
|---|---|
| A cloud provider | Its web console (Linode *Lish*, a serial console…), running the command below |
| A box you can sit at | Its own keyboard and screen |
| Some providers | The boot log they show you, which prints the SSH host keys |

The command, on the machine itself:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Compare it with what `ssh` showed you in step 3. Same? Keep it: you'll give it to `kvoadm` as
`--host-key`, so it checks the machine for you every time.

> **Why bother?** Everything after this (the cluster's CA, your admin certificate, secrets) is
> trusted because this first connection was to the right machine.

---

## Step 5 · Check the machine

Before changing anything, look. `kvoadm check` logs in exactly the way `init` will, reads the
machine, and **changes nothing**:

```bash
bin/kvoadm check root@203.0.113.10 --ssh-key ~/.ssh/kevaso_ed25519 --host-key SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4
```

```
root@203.0.113.10 (host key SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4)

✓ operating system    Ubuntu 24.04.1 LTS
✓ CPU                 x86_64            build kevasod with GOARCH=amd64
✓ account             root
✓ systemd             yes
✓ openssl             yes
✓ KVM                 /dev/kvm          microVM silos (cloud-hypervisor, openshell) can run here
✓ size                4 CPUs, 7953 MiB memory, 71 GiB free
! SSH password login  on                anyone can try passwords: set PasswordAuthentication no once your key works

Ready for kvoadm init. Nothing was changed.
```

| Mark | Means |
|---|---|
| `✓` | Fine |
| `!` | Works, but read the note |
| `✗` | Fix it before `init` (it exits non-zero) |

> **Password logins on?** Now that your key works, turn them off: on the machine, put
> `PasswordAuthentication no` in `/etc/ssh/sshd_config.d/00-local.conf` and run
> `systemctl reload ssh`. Keep your current session open until a new key login works.

## If something goes wrong

| Message | What to do |
|---|---|
| `host key mismatch` | The machine isn't the one you wrote down, or it was reinstalled. Check the fingerprint again (step 4). |
| `how should kvoadm log in?` | Give `--ssh-key PATH`, or `--password` to type it once. |
| `… has a passphrase: load it with ssh-add first` | Run `ssh-add ~/.ssh/kevaso_ed25519`, or run `kvoadm` in a terminal so it can ask. |
| `the host is aarch64 but --kevasod is built for x86_64` | Rebuild `kevasod` with the other `GOARCH` (step 1). |
| `host hv1 not Ready` | On the machine: `journalctl -u kevasod -n 50`. |
