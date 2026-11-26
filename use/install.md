---
title: Install kvo
permalink: /use/install/
---

# Install kvo

`kvo` is one program, for macOS, Linux and Windows. Take it from a
[release](https://github.com/dlebee/kevaso/releases): the macOS files are signed and notarized,
the Windows ones signed. You need nothing else on your computer to use a cluster.

Each step below is for your system: pick it in the tabs.

---

## 1 · Download it

With the [GitHub CLI](https://cli.github.com), into a folder of its own. Or take the same files
from the [release page](https://github.com/dlebee/kevaso/releases) in your browser: the archive for
your system, and `SHA256SUMS`.

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

```bash
gh release download v0.1.0-rc.11 --repo dlebee/kevaso --pattern 'kvo-*-darwin-arm64.zip' --pattern SHA256SUMS
```

That is for Apple Silicon. On an Intel Mac, take `darwin-amd64` instead of `darwin-arm64`, here
and below.

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
gh release download v0.1.0-rc.11 --repo dlebee/kevaso --pattern 'kvo-*-linux-amd64.tar.gz' --pattern SHA256SUMS
```

That is for x86_64. On ARM (`uname -m` says `aarch64`), take `linux-arm64` instead of
`linux-amd64`, here and below.

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

In PowerShell:

```powershell
gh release download v0.1.0-rc.11 --repo dlebee/kevaso --pattern 'kvo-*-windows-amd64.zip' --pattern SHA256SUMS
```

Windows builds are for x86_64 only.

</div>
</div>

## 2 · Check it

Compare the file with the release's checksums. A file that doesn't match: delete it and download
again.

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

```bash
shasum -a 256 --check --ignore-missing SHA256SUMS
```

It prints `kvo-v0.1.0-rc.11-darwin-arm64.zip: OK`.

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
sha256sum --check --ignore-missing SHA256SUMS
```

It prints `kvo-v0.1.0-rc.11-linux-amd64.tar.gz: OK`.

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

```powershell
Get-FileHash kvo-v0.1.0-rc.11-windows-amd64.zip
```

```powershell
Select-String windows-amd64 SHA256SUMS
```

The hash from the first must be the one on the `kvo-…` line of the second (the same letters and
digits, in another case).

</div>
</div>

## 3 · Unpack it

The archive unpacks into a folder of its own name, with `kvo` and its licence files.

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

```bash
unzip kvo-v0.1.0-rc.11-darwin-arm64.zip
```

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
tar -xzf kvo-v0.1.0-rc.11-linux-amd64.tar.gz
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

With the `tar` that comes with Windows:

```powershell
tar -xf kvo-v0.1.0-rc.11-windows-amd64.zip
```

</div>
</div>

## 4 · Put it on your PATH

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos" data-label="macOS" markdown="1">

`/usr/local/bin` is on the PATH of every macOS shell. Make sure it exists, then put `kvo` there
(both ask for your password):

```bash
sudo mkdir -p /usr/local/bin
```

```bash
sudo install -m 0755 kvo-v0.1.0-rc.11-darwin-arm64/kvo /usr/local/bin/kvo
```

</div>
<div class="os-tab" data-os="linux" data-label="Linux" markdown="1">

```bash
sudo install -m 0755 kvo-v0.1.0-rc.11-linux-amd64/kvo /usr/local/bin/kvo
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

Make a folder for it in your own profile, and move `kvo.exe` there:

```powershell
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Programs\Kevaso"
```

```powershell
Move-Item kvo-v0.1.0-rc.11-windows-amd64\kvo.exe "$env:LOCALAPPDATA\Programs\Kevaso"
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
kvo
```

It prints its usage line, starting `usage: kvo apply|get|describe|…`. A shell that says
`command not found` (or, on Windows, that `kvo` is not recognized) doesn't have the folder on its
PATH yet: open a new window.

---

## A new release

Download, check and unpack the new one the same way, and put it in the same place: it replaces the
old one.

---

## From source

From a checkout of [dlebee/kevaso](https://github.com/dlebee/kevaso), with Go:

<div class="os-tabs" markdown="1">
<div class="os-tab" data-os="macos linux" data-label="macOS, Linux" markdown="1">

```bash
go build -o bin/kvo ./cmd/kvo
```

</div>
<div class="os-tab" data-os="windows" data-label="Windows" markdown="1">

```powershell
go build -o bin\kvo.exe .\cmd\kvo
```

</div>
</div>

---

**Next:** [sign in]({{ '/use/sign-in/' | relative_url }}) with what your admin sent you.
