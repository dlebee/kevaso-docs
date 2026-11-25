---
title: Prepare a machine
permalink: /setup/prepare/
---

# Prepare a machine

Before Kevaso changes anything on a machine, check the machine runs a [supported operating system](#supported-operating-systems). Then you set up the way in, yourself, in four small steps:
build the tools, make an SSH key for Kevaso, put it on the machine, and write down the machine's
identity. You do this once for every machine: the system
node and each worker.

| Step | You do | Why |
|---|---|---|
| [1](#step-1--get-the-tools) | Get `kvo`, `kvoadm` and `kevasod` | From a release, or built from source |
| [2](#step-2--make-an-ssh-key-for-kevaso) | Make an SSH key for Kevaso | So `kvoadm` never needs a password |
| [3](#step-3--put-the-key-on-the-machine) | Put its public half on the machine | So the machine lets that key in |
| [4](#step-4--write-down-the-machines-fingerprint) | Write down the machine's host-key fingerprint | So you know you're talking to *your* machine |

Then: [install the control plane]({{ '/setup/single-node/' | relative_url }}) on it, or
[add it as a worker]({{ '/workers/' | relative_url }}).

---

## Supported operating systems

`kevasod` states which operating systems it supports. Today it lists:

| System | `ID` and `VERSION_ID` in `/etc/os-release` |
|---|---|
| Ubuntu 24.04 | `ubuntu` `24.04` |
| Debian 12 | `debian` `12` |
| macOS 26 (a [Mac worker]({{ '/workers/mac/' | relative_url }})) | `sw_vers -productVersion` 26 |

On the machine, `kevasod platforms` prints the list (`ubuntu 24.04, debian 12, macos 26`).

`kvoadm` reads the list out of the `kevasod` binary you give it with `--kevasod`, so the list it
judges by is the list of the binary it is about to install. If that binary has no list, `kvoadm`
uses the one built into itself.

Before it changes anything, `kvoadm init` and `kvoadm host add` read `/etc/os-release` on the
machine and compare the `ID` and `VERSION_ID` with the list, exactly:

- **On the list:** the line is `✓ operating system`.
- **Not on the list** (Ubuntu 22.04, Ubuntu 24.10, Debian 13, any other Linux): the line is a `!`
  warning, `this kevasod supports: ubuntu 24.04, debian 12, macos 26; others may work`. It is a warning, not a
  stop: `kvoadm` goes on.

The check stops (`✗`) for other reasons: a CPU that is not x86_64 or aarch64, no systemd, no
`openssl`, or something else already listening on port 6443. A machine without `apt-get` only gets a
warning, and then you install the runtimes' packages yourself.

---

## Step 1 · Get the tools

**From a release** (the simplest). Each [release](https://github.com/dlebee/kevaso/releases) has
`kvo` and `kvoadm` for macOS, Linux and Windows, and `kevasod` for Linux (x86_64 and ARM) and for
Apple Silicon Macs. The macOS files are signed and notarized, the Windows ones signed. Download the
ones you need with the GitHub CLI, into a folder of their own:

```bash
gh release download v0.1.0-rc.7 --repo dlebee/kevaso --pattern 'kvo*-darwin-arm64.zip' --pattern 'kevasod-*-linux-amd64.tar.gz' --pattern SHA256SUMS
```

Check them against the release's checksums, then unpack:

```bash
shasum -a 256 --check --ignore-missing SHA256SUMS
```

```bash
unzip kvo-v0.1.0-rc.7-darwin-arm64.zip && unzip kvoadm-v0.1.0-rc.7-darwin-arm64.zip && tar -xzf kevasod-v0.1.0-rc.7-linux-amd64.tar.gz
```

Each archive unpacks into a folder of its own name. Put the three programs in a `bin/` folder, as the
commands on these pages expect, and name `kevasod` for what it runs on:

```bash
mkdir -p bin && mv kvo-*/kvo kvoadm-*/kvoadm bin/ && mv kevasod-*-linux-amd64/kevasod bin/kevasod-linux-amd64
```

Pick the files for your computer (`darwin-arm64` for an Apple Silicon Mac, `linux-amd64`,
`windows-amd64`) and for each machine's CPU (`uname -m` on it says `x86_64` → `amd64`, or
`aarch64` → `arm64`). A [Mac worker]({{ '/workers/mac/' | relative_url }}) takes
`kevasod-…-darwin-arm64.zip`.

**From source.** From a checkout of [dlebee/kevaso](https://github.com/dlebee/kevaso), on your computer:

```bash
go build -o bin/kvo ./cmd/kvo
```

```bash
go build -o bin/kvoadm ./cmd/kvoadm
```

`kevasod` runs on the machine, so build it for Linux and for the machine's CPU (`amd64` for x86_64,
`arm64` for ARM):

```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags "-X kevaso.com/core/internal/version.Version=vX.Y.Z" -o bin/kevasod-linux-amd64 ./cmd/kevasod
```

> **Why `CGO_ENABLED=0`?** It makes a static binary that runs on any Linux, with nothing to install
> next to it.

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

> **Prefer not to do this by hand?** `kvoadm init --password` logs in with the password once and
> adds a key of its own for next time (`ssh/id_ed25519` in the cluster's folder, `~/.kevaso/clusters/CLUSTER/`). It changes nothing else about how
> the machine is logged in to.

---

## Step 4 · Write down the machine's fingerprint

The first time you connect to a machine, SSH shows its **host key fingerprint**, something like
`SHA256:kVvp5Yv9fNU4r4yX2P7+O99SU45jwdxC60ylimGFta4`. This is the machine's identity. If someone
sits between you and the machine, the fingerprint won't match, and that's the only moment you can
notice.

So get the real fingerprint **from somewhere other than this network connection**:

| Where the machine is | Where to read the fingerprint |
|---|---|
| A cloud provider | Its web console (a serial console…), running the command below |
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

## If something goes wrong

| Message | What to do |
|---|---|
| `host key not trusted: got SHA256:…, expected SHA256:…` | The machine isn't the one you wrote down, or it was reinstalled. Check the fingerprint again (step 4). |
| `how should kvoadm log in?` | Give `--ssh-key PATH`, or `--password` to type it once. |
| `… has a passphrase: load it with ssh-add first` | Run `ssh-add ~/.ssh/kevaso_ed25519`, or run `kvoadm` in a terminal so it can ask. |
| `the host is aarch64 but --kevasod is built for x86_64` | Take the `kevasod` for the other CPU from the release, or rebuild with the other `GOARCH` (step 1). |
| `host hv1 not Ready in …` | On the machine: `journalctl -u kevasod -n 50`. |
| `fix the ✗ lines first: nothing was changed` | The check of the machine found something that stops the install. Fix what its `✗` lines say (see above). |
