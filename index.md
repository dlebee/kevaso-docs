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
> [Get the tools]({{ '/setup/prepare/' | relative_url }}#step-1--get-the-tools)), or build them from source as the
> [quick start]({{ '/quickstart/' | relative_url }}) shows.

---

## Where to start

| If you want to | Go to |
|---|---|
| Understand the words first | [Concepts]({{ '/concepts/' | relative_url }}) |
| Try it on your own computer in 15 minutes | [Quick start]({{ '/quickstart/' | relative_url }}) |
| Decide how to set up a real cluster | [Choosing a setup]({{ '/setup/' | relative_url }}) |
| Set up a machine, step by step | [Prepare a machine]({{ '/setup/prepare/' | relative_url }}) |
| Run things inside a silo, or get a shell and files | [Daemons and upkeep]({{ '/silos/daemons/' | relative_url }}), [Terminals and files]({{ '/silos/terminals/' | relative_url }}), [Panes]({{ '/panes/' | relative_url }}) |
| Understand addresses, ports and names | [Addresses, ports and names]({{ '/networking/model/' | relative_url }}) |
| Keep a port closed until you need it | [Port leases]({{ '/networking/leases/' | relative_url }}) |
| Keep an address, or share one load balancer | [Addresses you keep]({{ '/networking/ips/' | relative_url }}) |
| Send traffic only to silos that are ready | [Probes]({{ '/silos/probes/' | relative_url }}) |
| Give a silo a secret without installing anything in it | [Secrets]({{ '/secrets/' | relative_url }}) |
| Give silos volumes, or make a microVM movable | [Volumes]({{ '/silos/volumes/' | relative_url }}), [Moves]({{ '/silos/moves/' | relative_url }}), [When the control plane is lost]({{ '/silos/control-plane-loss/' | relative_url }}) |
| Survive losing a system node, or upgrade without stopping | [High availability]({{ '/operations/ha/' | relative_url }}) |
| Add your own kinds and rules, or install a plugin | [Defining your own kind]({{ '/extending/kinds/' | relative_url }}), [Policies]({{ '/extending/policies/' | relative_url }}), [Plugins]({{ '/extending/plugins/' | relative_url }}), [Plugins for kvo]({{ '/extending/kvo-plugins/' | relative_url }}) |
| Look up a command | [Command reference]({{ '/reference/' | relative_url }}) |

---

## What's in a cluster

| | |
|---|---|
| **System node** | Runs the control plane: the API, the store, the scheduler, the sign-in provider, the secrets' master key. |
| **Workers** | Run silos. They only dial *out* to the system node, so a box behind your home router can be one. |
| **`kvo`** | The CLI for using it: create silos, step inside, manage secrets. |
| **`kvoadm`** | The CLI for running it: set up machines, invite people, rotate keys. |

## What you need

- **A Linux machine** for the system node: x86_64 or arm64, running a system `kevasod` lists as
  supported (Ubuntu 24.04 or Debian 12; [details]({{ '/setup/prepare/#supported-operating-systems' | relative_url }})).
  2 CPUs and 2 GiB of memory is plenty for the control plane alone.
- **KVM** (`/dev/kvm`) on the machines that run microVM silos: bare metal, or a VPS with nested
  virtualization. Without it, container silos still work.
- **Your computer** (macOS or Linux) with OpenSSH, and what the [quick start]({{ '/quickstart/' | relative_url }}) lists for building the programs.
