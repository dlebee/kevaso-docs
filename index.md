---
title: Overview
---

# Kevaso

Kevaso runs **silos** — isolated, long-lived machines (microVMs or containers) — on computers you
bring: a dedicated server, a VPS, a box under your desk. You describe what you want in YAML, and
Kevaso makes it so and keeps it that way, the way Kubernetes does for pods.

Unlike a pod, a silo is a *machine*: it has a disk that stays, a name, an address, and it can be the
place a developer works, or a service lives, for months. Its secrets can work inside it without ever
entering it.

> **Release candidates.** Kevaso is at `v0.1.0` release candidates: take `kvo`, `kvoadm` and
> `kevasod` from a [release](https://github.com/dlebee/kevaso/releases) (see
> [Install kvo]({{ '/use/install/' | relative_url }}) and [Install kvoadm]({{ '/admin/install/' | relative_url }})), or build them from source as the
> [quick start]({{ '/quickstart/' | relative_url }}) shows.

---

## Where to start

The docs have two parts, for two kinds of reader:

| You | You use | Start at |
|---|---|---|
| **Use the cluster**: make silos, step inside, give them secrets, ports and sites | `kvo`, with the credential an admin gave you | [Using Kevaso]({{ '/use/' | relative_url }}) |
| **Run the cluster**: set up machines, add hosts and people, back it up, upgrade it | `kvoadm`, over SSH to your machines | [Administration]({{ '/admin/' | relative_url }}) |

On your own cluster you are both. The [quick start]({{ '/quickstart/' | relative_url }}) plays both
parts on your own computer in 15 minutes, and [Concepts]({{ '/concepts/' | relative_url }}) has the
words. Every command is in the [command reference]({{ '/reference/' | relative_url }}).

---

## What's in a cluster

| | |
|---|---|
| **System node** | Runs the control plane: the API, the store, the scheduler, the sign-in provider, the secrets' master key. |
| **Workers** | Run silos. They only dial *out* to the system node, so a box behind your home router can be one. |
| **`kvo`** | The CLI for using it: create silos, step inside, manage secrets. [Using Kevaso]({{ '/use/' | relative_url }}) |
| **`kvoadm`** | The CLI for running it: set up machines, invite people, rotate keys. [Administration]({{ '/admin/' | relative_url }}) |

## What you need

- **A Linux machine** for the system node: x86_64 or arm64, running a system `kevasod` lists as
  supported (Ubuntu 24.04 or Debian 12; [details]({{ '/admin/setup/prepare/#supported-operating-systems' | relative_url }})).
  2 CPUs and 2 GiB of memory is plenty for the control plane alone.
- **KVM** (`/dev/kvm`) on the machines that run microVM silos: bare metal, or a VPS with nested
  virtualization. Without it, container silos still work.
- **Your computer** (macOS or Linux) with OpenSSH, and what the [quick start]({{ '/quickstart/' | relative_url }}) lists for building the programs.
