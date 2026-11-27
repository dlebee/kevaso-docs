---
title: Install kvoadm
permalink: /admin/install/
---

# Install kvoadm

An administrator needs two things on their own computer: `kvoadm`, and the `kevasod` that
`kvoadm` installs on the machines. `kvoadm` runs on macOS, Linux and Windows. `kevasod` runs on the
machines only: Linux (x86_64 and ARM) and Apple Silicon Macs, never Windows. Take them from a
[release](https://github.com/dlebee/kevaso/releases): the macOS files are signed and notarized, the
Windows ones signed.

`kvoadm kvo …` runs `kvo` commands as the cluster admin, and needs `kvo` too: install it the same
way ([Install kvo]({{ '/use/install/' | relative_url }})), or take both archives at once below.

Each step below is for your system: pick it in the tabs.

---

## 1 · Download it

With the [GitHub CLI](https://cli.github.com), into a folder of its own. Or take the same files
from the [release page](https://github.com/dlebee/kevaso/releases) in your browser. The pattern
`kvo*` takes both `kvo` and `kvoadm`.

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

```bash
gh release download v0.1.0-rc.13 --repo dlebee/kevaso --pattern 'kvo*-darwin-arm64.zip' --pattern SHA256SUMS
```

That is for Apple Silicon. On an Intel Mac, take `darwin-amd64` instead of `darwin-arm64`, here
and below.

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
gh release download v0.1.0-rc.13 --repo dlebee/kevaso --pattern 'kvo*-linux-amd64.tar.gz' --pattern SHA256SUMS
```

That is for x86_64. On ARM (`uname -m` says `aarch64`), take `linux-arm64` instead of
`linux-amd64`, here and below.

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

In PowerShell:

```powershell
gh release download v0.1.0-rc.13 --repo dlebee/kevaso --pattern 'kvo*-windows-amd64.zip' --pattern SHA256SUMS
```

Windows builds are for x86_64 only.

</div>
</div>

## 2 · Check it

Compare the files with the release's checksums. A file that doesn't match: delete it and download
again.

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

```bash
shasum -a 256 --check --ignore-missing SHA256SUMS
```

Each file it checks prints `: OK`.

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
sha256sum --check --ignore-missing SHA256SUMS
```

Each file it checks prints `: OK`.

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

```powershell
Get-FileHash kvo-v0.1.0-rc.13-windows-amd64.zip, kvoadm-v0.1.0-rc.13-windows-amd64.zip
```

```powershell
Select-String windows-amd64 SHA256SUMS
```

Each hash from the first must be the one on that file's line of the second (the same letters and
digits, in another case).

</div>
</div>

## 3 · Unpack it

Each archive unpacks into a folder of its own name.

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

```bash
unzip kvoadm-v0.1.0-rc.13-darwin-arm64.zip
```

```bash
unzip kvo-v0.1.0-rc.13-darwin-arm64.zip
```

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
tar -xzf kvoadm-v0.1.0-rc.13-linux-amd64.tar.gz
```

```bash
tar -xzf kvo-v0.1.0-rc.13-linux-amd64.tar.gz
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

With the `tar` that comes with Windows:

```powershell
tar -xf kvoadm-v0.1.0-rc.13-windows-amd64.zip
```

```powershell
tar -xf kvo-v0.1.0-rc.13-windows-amd64.zip
```

</div>
</div>

## 4 · Put it on your PATH

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

`/usr/local/bin` is on the PATH of every macOS shell. Make sure it exists, then put both programs
there (each asks for your password):

```bash
sudo mkdir -p /usr/local/bin
```

```bash
sudo install -m 0755 kvoadm-v0.1.0-rc.13-darwin-arm64/kvoadm /usr/local/bin/kvoadm
```

```bash
sudo install -m 0755 kvo-v0.1.0-rc.13-darwin-arm64/kvo /usr/local/bin/kvo
```

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
sudo install -m 0755 kvoadm-v0.1.0-rc.13-linux-amd64/kvoadm /usr/local/bin/kvoadm
```

```bash
sudo install -m 0755 kvo-v0.1.0-rc.13-linux-amd64/kvo /usr/local/bin/kvo
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

Make a folder for them in your own profile, and move both there:

```powershell
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Programs\Kevaso"
```

```powershell
Move-Item kvoadm-v0.1.0-rc.13-windows-amd64\kvoadm.exe, kvo-v0.1.0-rc.13-windows-amd64\kvo.exe "$env:LOCALAPPDATA\Programs\Kevaso"
```

Add that folder to your own PATH (not the system's: no administrator rights needed):

```powershell
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";$env:LOCALAPPDATA\Programs\Kevaso", "User")
```

Close PowerShell and open a new window, which reads the new PATH.

</div>
</div>

## 5 · Check it runs

```bash
kvoadm
```

With no command, it prints every command it has, starting `usage:` and `kvoadm init …`. A shell
that says `command not found` (or, on Windows, that `kvoadm` is not recognized) doesn't have the
folder on its PATH yet: open a new window.

The pages here write `bin/kvoadm` and `bin/kvo`, for programs kept in a `bin` folder. With them on
your PATH, type `kvoadm` and `kvo`.

**On Windows**, where a page says `~/.ssh/…`, type `$HOME\.ssh\…`. A command split over several
lines ends each line with `\`; in PowerShell end it with a backtick (`` ` ``) instead, or put it all
on one line.

---

## kevasod, for the machines

`kvoadm init` and `kvoadm host add` copy `kevasod` onto a machine from your computer
(`--kevasod PATH`). Take the build for the machine, not for your computer: `uname -m` on the
machine says `x86_64` (take `linux-amd64`) or `aarch64` (take `linux-arm64`). A
[Mac worker]({{ '/admin/workers/mac/' | relative_url }}) takes `kevasod-…-darwin-arm64.zip`.

Keep it in a `bin` folder, named for what it runs on, as the setup pages expect
(`--kevasod bin/kevasod-linux-amd64`). For x86_64 Linux machines:

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

```bash
gh release download v0.1.0-rc.13 --repo dlebee/kevaso --pattern 'kevasod-*-linux-amd64.tar.gz' --pattern SHA256SUMS --clobber
```

```bash
shasum -a 256 --check --ignore-missing SHA256SUMS
```

```bash
tar -xzf kevasod-v0.1.0-rc.13-linux-amd64.tar.gz
```

```bash
mkdir -p bin
```

```bash
mv kevasod-v0.1.0-rc.13-linux-amd64/kevasod bin/kevasod-linux-amd64
```

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
gh release download v0.1.0-rc.13 --repo dlebee/kevaso --pattern 'kevasod-*-linux-amd64.tar.gz' --pattern SHA256SUMS --clobber
```

```bash
sha256sum --check --ignore-missing SHA256SUMS
```

```bash
tar -xzf kevasod-v0.1.0-rc.13-linux-amd64.tar.gz
```

```bash
mkdir -p bin
```

```bash
mv kevasod-v0.1.0-rc.13-linux-amd64/kevasod bin/kevasod-linux-amd64
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

```powershell
gh release download v0.1.0-rc.13 --repo dlebee/kevaso --pattern 'kevasod-*-linux-amd64.tar.gz' --pattern SHA256SUMS --clobber
```

```powershell
Get-FileHash kevasod-v0.1.0-rc.13-linux-amd64.tar.gz
```

```powershell
Select-String kevasod SHA256SUMS
```

```powershell
tar -xf kevasod-v0.1.0-rc.13-linux-amd64.tar.gz
```

```powershell
New-Item -ItemType Directory -Force bin
```

```powershell
Move-Item kevasod-v0.1.0-rc.13-linux-amd64\kevasod bin\kevasod-linux-amd64
```

</div>
</div>

A new release is rolled over the cluster with `kvoadm upgrade`, one `--kevasod` per operating
system and CPU the machines have ([Upgrade]({{ '/admin/operations/ha/#upgrade' | relative_url }})).
Install the new `kvoadm` and `kvo` first, the same way as above.

---

## From source

From a checkout of [dlebee/kevaso](https://github.com/dlebee/kevaso), with Go:

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos linux" data-label="macOS, Linux" markdown="1">

```bash
go build -o bin/kvoadm ./cmd/kvoadm
```

```bash
go build -o bin/kvo ./cmd/kvo
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
go build -o bin\kvoadm.exe .\cmd\kvoadm
```

```powershell
go build -o bin\kvo.exe .\cmd\kvo
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

**Next:** [Prepare a machine]({{ '/admin/setup/prepare/' | relative_url }}).
