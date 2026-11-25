---
title: A Mac as a worker
permalink: /workers/mac/
---

# A Mac as a worker

An Apple Silicon Mac can join a cluster as a worker and run **macOS silos**: macOS virtual machines,
for building iOS and macOS apps and running the iOS Simulator. It runs `kevasod` like any worker, and
one runtime, [`tart`]({{ '/silos/runtimes/' | relative_url }}#tart-macos-silos). The system node stays
on Linux.

| | |
|---|---|
| **The Mac** | Apple Silicon, macOS 26 (on `kevasod`'s [supported list]({{ '/setup/prepare/#supported-operating-systems' | relative_url }})) |
| **Silos** | At most **two** running at a time per Mac: Apple's licence for macOS allows two virtual copies. More Macs, more macOS silos |
| **Where** | Anywhere: on your LAN, at home behind NAT, on another network. It dials out to the system node |
| **Not on a Mac** | MicroVMs, containers, volumes, ports and services for its silos. A Mac is never a system node |

---

## Prepare the Mac

Do this once, on the Mac.

1. **Turn on Remote Login:** System Settings → General → Sharing → Remote Login.
2. **Put your Kevaso key on it**, for the Mac's own user, the one that will run the silos. Not root:
   Tart's images and VMs belong to a user. The steps are on [Prepare a machine]({{ '/setup/prepare/' | relative_url }});
   the fingerprint command is the same on a Mac.
3. **Let that user run `sudo` without a password.** `kvoadm` makes every change with `sudo -n`, and
   `kevasod` starts `softnet` (below) through `sudo`. Run `sudo visudo -f /etc/sudoers.d/kevaso` and
   add one line, with the user's name:

   ```text
   USER ALL=(ALL) NOPASSWD: ALL
   ```

4. **Install Tart, softnet and OpenSSL 3** with [Homebrew](https://brew.sh):

   ```bash
   brew install openai/tools/tart openai/tools/softnet openssl@3
   ```

   | | Why |
   |---|---|
   | `tart` | Runs the macOS VMs. Licensed FSL-1.1-ALv2: Kevaso installs nothing of it and only runs the `tart` program you installed |
   | `softnet` | Keeps each silo off your LAN and away from the other silos (the internet is allowed). Without it, silos reach each other and every machine on your network |
   | `openssl@3` | The Mac checks `kevasod`'s signature with it: macOS's own `openssl` can't read the key |

5. **Pull the image your silos will use.** It is 25 to 70 GB, so do it ahead; a silo started
   without it waits for the download:

   ```bash
   tart pull ghcr.io/cirruslabs/macos-tahoe-base:latest
   ```

---

## Add it

From your computer, with the `kevasod` for macOS from a [release]({{ '/setup/prepare/#step-1--get-the-tools' | relative_url }})
(`kevasod-…-darwin-arm64.zip`, signed and notarized):

```bash
bin/kvoadm host add USER@192.0.2.50 \
    --ssh-key ~/.ssh/kevaso_ed25519 \
    --host-key SHA256:… \
    --kevasod bin/kevasod-darwin-arm64 \
    --internal-ip 192.168.1.50 \
    --behind-nat \
    --name mac1
```

| Flag | Meaning |
|---|---|
| `USER@192.0.2.50` | The Mac's own user, at an address you reach it on: on your LAN, or over a VPN such as Tailscale when it's elsewhere |
| `--internal-ip` | The Mac's address on its own network (`ipconfig getifaddr en0`) |
| `--behind-nat` | Nothing reaches it from outside (a Mac at home): it dials out, and reaches the other hosts through a relay ([How workers connect]({{ '/workers/' | relative_url }})) |

It looks first, and says what it found (`--dry-run` stops there):

```text
What's on this machine:
  ✓ operating system    macOS 26.6.2
  ✓ CPU                 arm64              Apple Silicon: build kevasod with GOOS=darwin GOARCH=arm64
  ✓ account             uid 501 (USER)     tart runs as this user; kvoadm runs every change with sudo -n
  ✓ launchd             yes
  ✓ openssl             3 or later
  ✓ tart                2.42.0             macOS silos: at most two running per Mac (Apple's licence)
  ✓ softnet             installed          needs a sudoers rule for that binary, or kevasod's --tart-softnet=false
  ✓ size                10 CPUs, 16384 MiB memory, 126 GiB free   a macOS silo is 4 CPUs and 8 GiB by default, and its image is 25-70 GB: plan 150 GB free
  ✓ supported           macOS 26           on kevasod's supported list
```

Then it installs two launchd daemons, `com.kevaso.kevasod` (`kevasod` as root, `tart` as your user)
and `com.kevaso.fence`, and the Mac joins. A minute later:

```bash
bin/kvoadm kvo get hosts
```

```text
NAME   INTERNAL IP    EXTERNAL IP   REGION   COMPONENTS   RUNTIMES     AGE
mac1   192.168.1.50   -             -        worker       tart:Ready   1m
```

> **A firewall in front of the system node?** It must let in UDP 51820 (the mesh) from the Mac's
> public address, as from every worker, or the Mac never joins the mesh.

---

## A macOS silo

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: mac, namespace: demo }
spec:
  runtime: tart
  container: { image: ghcr.io/cirruslabs/macos-tahoe-base:latest }
  resources: { cpu: 4, memory: 8Gi }
```

```bash
bin/kvo apply -f mac.yaml
```

```bash
bin/kvo exec silo mac -n demo -- sw_vers
```

It is `Running` about 30 seconds after `apply` when the image is already on the Mac. What it can and
can't do is on [Runtimes]({{ '/silos/runtimes/' | relative_url }}#tart-macos-silos).

---

## Upgrades, a new system node, removing it

| | |
|---|---|
| **Upgrade** | Give `kvoadm upgrade` a build for each kind of host: `--kevasod bin/kevasod-linux-amd64 --kevasod bin/kevasod-darwin-arm64`. Each host gets its own, and a cluster with a Mac but no macOS build is refused before anything changes ([High availability]({{ '/operations/ha/#upgrade' | relative_url }})) |
| **A restored system node** | `kvoadm init --restore` points the Mac at the new address over SSH, as it does every worker ([Backup and restore]({{ '/operations/backup/' | relative_url }})) |
| **Remove** | `bin/kvoadm reset mac1` removes Kevaso's daemons and files. It never touches Tart, its images or SSH. A Mac still running macOS silos is refused; `--force` removes their VMs too |

`host add` also puts the cluster's admin SSH keys in the user's `~/.ssh/authorized_keys`, between
`# kevaso-admins begin` and `# kevaso-admins end`. `reset` leaves SSH alone: delete those lines
yourself if the Mac leaves for good.
