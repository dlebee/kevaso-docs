---
title: Concepts
permalink: /concepts/
---

# Concepts

## Silo

A silo is the unit Kevaso runs: one stateful machine, either a microVM or a container. Silos are
never replaced by copies; Kevaso enforces their desired state in place. A silo lives on one host.

Its `runtime` picks the implementation:

| Family | Runtimes |
|---|---|
| microVM | `cloud-hypervisor`, `openshell` |
| container | `docker`, `podman`, `openshell-container` |

Operators choose which runtimes each worker offers. microVM runtimes need KVM.

## Host

A host is a machine you bring, plus an SSH key or initial root password. `kvoadm` sets it up once
over SSH; it never creates machines itself. Any modern Linux distribution with systemd (and KVM, for
microVMs) works.

## Components

A host runs the components you install on it, and nothing is hidden:

- **control-plane**: `kvoadm init` installs it. A host with it is a *system node*.
- **worker**: `kvoadm host install <host> worker` adds it. A host with it is a *worker node*.

## Namespaces

Namespaces span hosts. Silos deny traffic from each other by default, even in the same namespace.
