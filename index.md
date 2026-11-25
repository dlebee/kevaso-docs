---
title: Introduction
description: What Kevaso is, the ideas it is built on, and who does what with it.
---

# Kevaso

Kevaso runs **silos**, isolated and long-lived machines, on computers you bring, and manages them
the way Kubernetes manages pods: you describe what you want, and Kevaso makes it so and keeps it
that way.

> **Release candidates.** Kevaso is at `v0.1.0` release candidates. The programs are on the
> [releases page](https://github.com/dlebee/kevaso/releases).

---

## The idea

A Kubernetes pod is disposable: it has no disk of its own that outlives it, and it can be replaced
at any moment. Many things don't fit that shape. A developer's workspace, a database, a build
machine, a sandbox an agent works in: each needs a disk that stays, a name, an address, and a life
measured in months.

A silo is that kind of machine. It is a **microVM** (with its own kernel), a **container**, an
**OpenShell sandbox**, or a **macOS virtual machine** on a Mac, and it keeps its disk across stops,
restarts and moves. You still work with it declaratively: a silo is a YAML object, and `kvo apply`
is how you make or change one.

Kevaso runs on machines you own or rent: a dedicated server, a virtual server, a computer under a
desk. There is no cloud service behind it.

---

## The main ideas

### Silos

The thing you make. A silo has a runtime (what it is made of), resources, a root disk, and
optionally volumes, init steps that prepare it, programs Kevaso keeps running in it, and probes that
say when it is ready. A silo with none of the optional parts is still a normal machine: Kevaso
starts it, keeps it started, and otherwise leaves it alone. Nothing of Kevaso's is installed inside
a silo.

### Hosts: the control plane and the workers

Each machine Kevaso manages is a **host**, running one program, `kevasod`.

- The **control plane** stores what you asked for, checks who may do what, and decides which host
  each silo runs on. It can run on one machine, or on several for high availability.
- **Workers** run silos. A worker only ever dials *out* to the control plane, so it needs no open
  port: a machine behind a home router can be one.

If a worker loses the control plane, nothing on it changes: its silos keep running as they were.

### Namespaces

Silos and the things they use (secrets, grants, services) live in a **namespace**. Access is given
per namespace: a person can make and change silos in the namespaces they were let into, and sees no
other.

### The mesh

Hosts talk to each other over an encrypted network of their own (WireGuard), so silos on different
machines, in different places, reach each other at cluster addresses and by name.

### Default deny, and Grants

A new silo reaches nothing outside the cluster, and nothing reaches it. Every opening is a rule you
can read:

- A silo **declares its ports**, and a **service** opens each one to the silos you name.
- A **Grant** says what a silo may reach outside the cluster, and which secrets it gets.

### Secrets that never enter a silo

A secret (an API key, a password, a signing key) is kept encrypted by the control plane. A silo gets
it only through a Grant, and the Grant decides how. With **injection**, the value is added to the
silo's requests to one destination, outside the silo. With **signing**, the key stays in Kevaso and
the silo gets signatures. Either way the value never enters the silo. When a program needs the value itself, a Grant can hand it
over on request, or as a file or an environment variable.

### Moves

A silo is placed on one host, but it is not tied to it. A silo whose storage can be reached from
other hosts can **move** to another host with its data, by your command or when its host is lost.
A container that keeps nothing of its own starts fresh elsewhere instead.

---

## Two roles, two programs

| | People who use Kevaso | Administrators who run it |
|---|---|---|
| Do | Make silos, step inside, give them ports, volumes, secrets and sites | Set up machines, add hosts and people, back up, upgrade |
| With | [`kvo`]({{ '/use/' | relative_url }}), like `kubectl` | [`kvoadm`]({{ '/admin/' | relative_url }}), like `kubeadm` |
| Get in | With the access an administrator gave them | Over SSH to the machines, to set them up |
| Start at | [Getting started with kvo]({{ '/use/' | relative_url }}) | [Administering Kevaso]({{ '/admin/' | relative_url }}) |

Most people are users: an administrator lets them into a namespace, and from then on they need
only `kvo`. On a cluster of your own you are both.

---

## Where next

- [Concepts]({{ '/concepts/' | relative_url }}): the words, the picture of a cluster, and how a change happens.
- [Getting started with kvo]({{ '/use/' | relative_url }}): you have access to a cluster, and want your first silo.
- [Administering Kevaso]({{ '/admin/' | relative_url }}): you want to set up a cluster of your own.
- [Command reference]({{ '/reference/' | relative_url }}): every command.
