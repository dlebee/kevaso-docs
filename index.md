---
title: Overview
---

# Kevaso

Kevaso runs self-hosted, stateful **silos** (microVMs or containers) on the machines you bring,
managed Kubernetes-style. One cluster can mix bare metal, any cloud and home machines.

> **Status: early.** Kevaso is not ready to use yet. These docs grow as the
> [milestones](https://github.com/dlebee/kevaso/blob/main/MILESTONES.md) land.

## The commands

| Command | For |
|---|---|
| `kvo` (or `kevaso`) | users: create, inspect and enter silos |
| `kvoadm` | operators: set up hosts and the cluster |
| `kevasod` | the agent that runs on every host |

## Next

- [Concepts](concepts/): silos, hosts and components.
- [Contributing](contributing/): build and test the code.
