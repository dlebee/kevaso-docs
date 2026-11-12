---
title: High availability
permalink: /setup/ha/
---

# High availability <span class="badge planned">planned (M16)</span>

**Not available yet.** Today a cluster has one system node. This page describes what's planned, so
you can choose machines with it in mind. Its parts are proven in research (embedded etcd, SQLite
migration) but not built into Kevaso.

---

## What it will give you

Several system nodes share the cluster's state, and any of them serves the API. Losing one of them
changes nothing for you: no failover to run, no address to move.

| | Today (one system node) | With HA |
|---|---|---|
| A system node fails | No changes until it's back; `Recreate` silos stop after 30 s | Nothing visible |
| Upgrading the control plane | A short API outage | One node at a time, no outage |
| Store | SQLite on one disk | Embedded etcd (Raft), on every system node |

---

## How many system nodes

The store uses **Raft**, which needs a majority of its members to accept any change:

| System nodes | Can lose | Notes |
|---|---|---|
| 1 | 0 | Today |
| 2 | 0 | **Worse than one**: either one failing stops all changes. Only as a step on the way to 3. |
| 3 | 1 | The usual choice |
| 5 | 2 | For clusters spread over sites |

**Only two real machines?** The third vote can be a **light system node**: it holds the state and
votes, with no API, controllers or silos (about half a CPU and 512 MiB, but a steady network and a
decent disk, since a slow member slows every write). Put it in a third place (a small VPS, the box at
home) so one site can't hold two votes.

---

## How it will be set up

Planned: adding the control plane to a second and a third machine, each prepared like the first
([Prepare a machine]({{ '/setup/prepare/' | relative_url }})). The store moves from SQLite to
embedded etcd by itself on the way. Workers already find the control plane through the mesh, so no
load balancer or floating address is needed: each worker spreads its connections over the system
nodes itself.

---

## Operational risks to plan for

- **Three machines to keep healthy.** One down is fine; two down stops changes (running silos keep
  running, as with one system node today).
- **Disk and network matter.** Every write waits for a majority; one slow member slows the cluster.
- **Spread them.** Three system nodes in one rack fail together. Different machines, ideally different
  sites.
- **Secrets everywhere.** Every system node holds the master key for secrets, so every system node
  needs the care the first one gets.
- **Backups still matter.** HA survives a machine; a backup (and the recovery key) survives a mistake.
  Backup and restore come with the same milestone.
