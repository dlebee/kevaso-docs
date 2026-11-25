---
title: Prepare a machine
permalink: /admin/setup/prepare/
redirect_from: /setup/prepare/
---

# Prepare a machine

Before Kevaso changes anything on a machine, check the machine runs a [supported operating system](#supported-operating-systems). Then you set up the way in, yourself, in four small steps:
get the tools, make an SSH key for Kevaso, put it on the machine, and write down the machine's
identity. You do this once for every machine: the system
node and each worker.

| Step | You do | Why |
|---|---|---|
| [1](#step-1--get-the-tools) | Get `kvo`, `kvoadm` and `kevasod` | From a release, or built from source |
| [2](#step-2--make-an-ssh-key-for-kevaso) | Make an SSH key for Kevaso | So `kvoadm` never needs a password |
| [3](#step-3--put-the-key-on-the-machine) | Put its public half on the machine | So the machine lets that key in |
| [4](#step-4--write-down-the-machines-fingerprint) | Write down the machine's host-key fingerprint | So you know you're talking to *your* machine |

Then: [install the control plane]({{ '/admin/setup/single-node/' | relative_url }}) on it, or
[add it as a worker]({{ '/admin/workers/' | relative_url }}).

---

## Supported operating systems

`kevasod` states which operating systems it supports. Today it lists:

| System | `ID` and `VERSION_ID` in `/etc/os-release` |
|---|---|
| Ubuntu 24.04 | `ubuntu` `24.04` |
| Debian 12 | `debian` `12` |
| macOS 26 (a [Mac worker]({{ '/admin/workers/mac/' | relative_url }})) | `sw_vers -productVersion` 26 |

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
Apple Silicon Macs. The macOS files are signed and notarized, the Windows ones signed.

Your computer needs `kvo` and `kvoadm` for its own system; the machines need `kevasod` for theirs
(`uname -m` on a machine says `x86_64` → `linux-amd64`, or `aarch64` → `linux-arm64`; a
[Mac worker]({{ '/admin/workers/mac/' | relative_url }}) takes `kevasod-…-darwin-arm64.zip`). The commands
below fetch `kevasod` for x86_64 Linux machines. Download into a folder of their own, with the
[GitHub CLI](https://cli.github.com):

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

```bash
gh release download v0.1.0-rc.7 --repo dlebee/kevaso --pattern 'kvo*-darwin-arm64.zip' --pattern 'kevasod-*-linux-amd64.tar.gz' --pattern SHA256SUMS
```

On an Intel Mac, take `darwin-amd64` instead of `darwin-arm64`, here and below. Check the files
against the release's checksums:

```bash
shasum -a 256 --check --ignore-missing SHA256SUMS
```

Unpack them. Each archive unpacks into a folder of its own name:

```bash
unzip kvo-v0.1.0-rc.7-darwin-arm64.zip && unzip kvoadm-v0.1.0-rc.7-darwin-arm64.zip && tar -xzf kevasod-v0.1.0-rc.7-linux-amd64.tar.gz
```

Put the three programs in a `bin/` folder, as the commands on these pages expect, and name
`kevasod` for what it runs on:

```bash
mkdir -p bin && mv kvo-*/kvo kvoadm-*/kvoadm bin/ && mv kevasod-*-linux-amd64/kevasod bin/kevasod-linux-amd64
```

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
gh release download v0.1.0-rc.7 --repo dlebee/kevaso --pattern 'kvo*-linux-amd64.tar.gz' --pattern 'kevasod-*-linux-amd64.tar.gz' --pattern SHA256SUMS
```

On an ARM computer, take `kvo*-linux-arm64.tar.gz` for your own tools. Check the files against
the release's checksums:

```bash
sha256sum --check --ignore-missing SHA256SUMS
```

Unpack them. Each archive unpacks into a folder of its own name:

```bash
tar -xzf kvo-v0.1.0-rc.7-linux-amd64.tar.gz && tar -xzf kvoadm-v0.1.0-rc.7-linux-amd64.tar.gz && tar -xzf kevasod-v0.1.0-rc.7-linux-amd64.tar.gz
```

Put the three programs in a `bin/` folder, as the commands on these pages expect, and name
`kevasod` for what it runs on:

```bash
mkdir -p bin && mv kvo-*/kvo kvoadm-*/kvoadm bin/ && mv kevasod-*-linux-amd64/kevasod bin/kevasod-linux-amd64
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

In PowerShell:

```powershell
gh release download v0.1.0-rc.7 --repo dlebee/kevaso --pattern 'kvo*-windows-amd64.zip' --pattern 'kevasod-*-linux-amd64.tar.gz' --pattern SHA256SUMS
```

Check each file's hash against its line in `SHA256SUMS` (the same letters and digits, in another
case):

```powershell
Get-FileHash kvo-v0.1.0-rc.7-windows-amd64.zip, kvoadm-v0.1.0-rc.7-windows-amd64.zip, kevasod-v0.1.0-rc.7-linux-amd64.tar.gz
```

```powershell
Get-Content SHA256SUMS
```

Unpack them with the `tar` that comes with Windows. Each archive unpacks into a folder of its own
name:

```powershell
tar -xf kvo-v0.1.0-rc.7-windows-amd64.zip
```

```powershell
tar -xf kvoadm-v0.1.0-rc.7-windows-amd64.zip
```

```powershell
tar -xf kevasod-v0.1.0-rc.7-linux-amd64.tar.gz
```

Put the three programs in a `bin` folder, and name `kevasod` for what it runs on:

```powershell
mkdir bin
```

```powershell
Move-Item kvo-v0.1.0-rc.7-windows-amd64\kvo.exe, kvoadm-v0.1.0-rc.7-windows-amd64\kvoadm.exe bin
```

```powershell
Move-Item kevasod-v0.1.0-rc.7-linux-amd64\kevasod bin\kevasod-linux-amd64
```

**The commands on these pages, in PowerShell:** type `bin\kvoadm.exe` and `bin\kvo.exe` where
they say `bin/kvoadm` and `bin/kvo`, and `$HOME\.ssh\…` where they say `~/.ssh/…`. A command split
over several lines ends each line with `\`; in PowerShell end it with a backtick (`` ` ``)
instead, or put it all on one line.

</div>
</div>

**From source.** From a checkout of [dlebee/kevaso](https://github.com/dlebee/kevaso), with Go:

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos linux" data-label="macOS, Linux" markdown="1">

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

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

In PowerShell:

```powershell
go build -o bin\kvo.exe .\cmd\kvo
```

```powershell
go build -o bin\kvoadm.exe .\cmd\kvoadm
```

`kevasod` runs on the machine, so build it for Linux and for the machine's CPU (`amd64` for x86_64,
`arm64` for ARM). Set the target for this PowerShell window, then build:

```powershell
$env:GOOS = "linux"; $env:GOARCH = "amd64"; $env:CGO_ENABLED = "0"
```

```powershell
go build -ldflags "-X kevaso.com/core/internal/version.Version=vX.Y.Z" -o bin\kevasod-linux-amd64 .\cmd\kevasod
```

Close the window afterwards (or set `$env:GOOS = ""` and `$env:GOARCH = ""`), so later builds are
for Windows again.

</div>
</div>

> **Why `CGO_ENABLED=0`?** It makes a static binary that runs on any Linux, with nothing to install
> next to it.

---

## Step 2 · Make an SSH key for Kevaso

`kvoadm` logs in to your machines over SSH to install `kevasod`. Give it a key of its own, separate
from your everyday key:

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos linux" data-label="macOS, Linux" markdown="1">

```bash
ssh-keygen -t ed25519 -f ~/.ssh/kevaso_ed25519 -C "kevaso admin"
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

In PowerShell, with the OpenSSH client that comes with Windows:

```powershell
ssh-keygen -t ed25519 -f $HOME\.ssh\kevaso_ed25519 -C "kevaso admin"
```

On Windows the files below are in `$HOME\.ssh\` (`C:\Users\YOU\.ssh\`).

</div>
</div>

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

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos linux" data-label="macOS, Linux" markdown="1">

```bash
cat ~/.ssh/kevaso_ed25519.pub
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

```powershell
Get-Content $HOME\.ssh\kevaso_ed25519.pub
```

</div>
</div>

**B · A machine you can already log in to with a password** — copy the key over with OpenSSH:

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos linux" data-label="macOS, Linux" markdown="1">

```bash
ssh-copy-id -i ~/.ssh/kevaso_ed25519.pub root@203.0.113.10
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

Windows has no `ssh-copy-id`, so send the key over `ssh` and add it on the machine:

```powershell
Get-Content $HOME\.ssh\kevaso_ed25519.pub | ssh root@203.0.113.10 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

</div>
</div>

This is the one time you type the password. Keep reading before you answer `yes` to the question it
asks: that's step 4.

Then check that the key works on its own:

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos linux" data-label="macOS, Linux" markdown="1">

```bash
ssh -i ~/.ssh/kevaso_ed25519 root@203.0.113.10
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

```powershell
ssh -i $HOME\.ssh\kevaso_ed25519 root@203.0.113.10
```

</div>
</div>

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
