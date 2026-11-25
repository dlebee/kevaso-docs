---
title: Administering Kevaso
permalink: /admin/
---

# Administering Kevaso

This part is for the people who run a cluster: you set up the machines and the control plane, add
and reset hosts, let people in, back it up and upgrade it. You do it with `kvoadm`, from your own
computer.

---

## How an administrator works

- **`kvoadm` logs in to your machines over SSH** to install and change `kevasod`. Nothing else
  needs SSH: once a machine is set up, everything goes through the API.
- **Each cluster is a folder**, `~/.kevaso/clusters/NAME/`: a `cluster.yaml` the admins share in a
  repository, and a `local.yaml` that stays on your computer
  ([kvoadm's cluster folder]({{ '/admin/access/#kvoadms-cluster-folder' | relative_url }})).
- **The admin certificate stays in that folder.** Your own `kvo` is a plain `editor`, like
  everyone else's. A step a user page marks as an admin's, you run as `kvoadm kvo …`
  ([Your own kvo, and the admin certificate]({{ '/admin/access/#your-own-kvo-and-the-admin-certificate' | relative_url }})).

---

## The pages

| If you want to | Go to |
|---|---|
| Install `kvoadm`, `kvo` and `kevasod` on your computer | [Install kvoadm]({{ '/admin/install/' | relative_url }}) |
| Decide how to set up a real cluster | [Choosing a setup]({{ '/admin/setup/' | relative_url }}) |
| Set up a machine, then the control plane | [Prepare a machine]({{ '/admin/setup/prepare/' | relative_url }}), [Single system node]({{ '/admin/setup/single-node/' | relative_url }}) |
| Try it in a VM on your computer | [Running locally]({{ '/admin/setup/local/' | relative_url }}) |
| Add workers, a Mac among them, or reset a machine | [Adding workers]({{ '/admin/workers/' | relative_url }}), [A Mac as a worker]({{ '/admin/workers/mac/' | relative_url }}) |
| Drain or remove a host | [Draining and removing a host]({{ '/admin/workers/host-loss/' | relative_url }}) |
| Know what happens when a worker loses the control plane | [When the control plane is lost]({{ '/admin/control-plane-loss/' | relative_url }}) |
| Invite people, give roles, make CI tokens, add admins | [People and access]({{ '/admin/access/' | relative_url }}) |
| Reserve addresses, add extra addresses to a host | [Addresses you keep]({{ '/admin/ips/' | relative_url }}) |
| Set up an ingress controller, its class and issuers | [Ingress controllers and certificates]({{ '/admin/ingress/' | relative_url }}) |
| Find the logs, the store and the keys | [Operations]({{ '/admin/operations/' | relative_url }}) |
| Survive losing a system node, or upgrade without stopping | [High availability]({{ '/admin/operations/ha/' | relative_url }}) |
| Back up and restore | [Backup and restore]({{ '/admin/operations/backup/' | relative_url }}) |
| Use a cloud provider's balancers, volumes and DNS | [Linode]({{ '/admin/providers/linode/' | relative_url }}), [JuiceFS]({{ '/admin/providers/juicefs/' | relative_url }}) |
| Add your own kinds and rules, or install a plugin | [Defining your own kind]({{ '/admin/extending/kinds/' | relative_url }}), [Policies]({{ '/admin/extending/policies/' | relative_url }}), [Provider and operator plugins]({{ '/admin/extending/plugins/' | relative_url }}) |
| Look up a command | [kvoadm and kevasod commands]({{ '/reference/kvoadm/' | relative_url }}) |
