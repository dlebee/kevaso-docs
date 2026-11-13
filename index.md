---
title: Overview
---

# Kevaso

Kevaso runs **silos** — isolated, long-lived machines (microVMs or containers) — on computers you
bring: a dedicated server, a VPS, a box under your desk. You describe what you want in YAML, and
Kevaso makes it so and keeps it that way, the way Kubernetes does for pods.

Unlike a pod, a silo is a *machine*: it has a disk that stays, a name, an address, and it can be the
place an AI agent or a developer works for months. Its secrets can work inside it without ever
entering it.

> **Built from source.** Kevaso has no packaged releases: you build `kvo`, `kvoadm` and `kevasod`
> with Go, as the [quick start]({{ '/quickstart/' | relative_url }}) shows.

---

## Where to start

| If you want to | Go to |
|---|---|
| Understand the words first | [Concepts]({{ '/concepts/' | relative_url }}) |
| Try it on your own computer in 15 minutes | [Quick start]({{ '/quickstart/' | relative_url }}) |
| Decide how to set up a real cluster | [Choosing a setup]({{ '/setup/' | relative_url }}) |
| Set up a machine, step by step | [Prepare a machine]({{ '/setup/prepare/' | relative_url }}) |
| Run things inside a silo, or get a shell and files | [Daemons and upkeep]({{ '/silos/daemons/' | relative_url }}), [Terminals and files]({{ '/silos/terminals/' | relative_url }}), [Panes]({{ '/panes/' | relative_url }}) |
| Keep a port closed until you need it | [Port leases]({{ '/networking/leases/' | relative_url }}) |
| Give silos volumes, or make a microVM movable | [Volumes]({{ '/silos/volumes/' | relative_url }}), [Host loss and moves]({{ '/silos/moves/' | relative_url }}) |
| Survive losing a system node, or upgrade without stopping | [High availability]({{ '/operations/ha/' | relative_url }}) |
| Add your own kinds and rules, or install a plugin | [Defining your own kind]({{ '/extending/kinds/' | relative_url }}), [Policies]({{ '/extending/policies/' | relative_url }}), [Plugins]({{ '/extending/plugins/' | relative_url }}) |
| See what has been proven on real machines | [What's proven where]({{ '/operations/#whats-proven-where' | relative_url }}) |
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

- **A Linux machine** for the system node: Ubuntu 24.04, x86_64 or arm64. 2 CPUs and 2 GiB of
  memory is plenty for the control plane alone.
- **KVM** (`/dev/kvm`) on the machines that run microVM silos: bare metal, or a VPS with nested
  virtualization. Without it, container silos still work.
- **Your computer** (macOS or Linux) with Go 1.27 and OpenSSH.
