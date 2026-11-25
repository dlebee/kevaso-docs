---
title: Using Kevaso
permalink: /use/
---

# Using Kevaso

This part is for the people who use a cluster: you make silos, step inside them, give them
volumes, secrets, ports and HTTPS sites, all with `kvo`. An admin set up the machines and gave you
a way in; you never need SSH to a machine, nor `kvoadm`.

---

## What you need

- **`kvo` on your computer:** [Install kvo]({{ '/use/install/' | relative_url }}), with steps for
  macOS, Linux and Windows.
- **A way in, from an admin:** a `kvo login …` line and a one-time code, for a namespace. Then
  [sign in]({{ '/use/sign-in/' | relative_url }}).

Your role decides what you may do. Most people are an `editor` in their namespaces: everything on
these pages except the steps marked as an admin's. `kvo auth can-i` answers for any one step.

---

## The pages

| If you want to | Go to |
|---|---|
| Install `kvo` | [Install kvo]({{ '/use/install/' | relative_url }}) |
| Sign in, switch clusters, see what you may do | [Signing in and contexts]({{ '/use/sign-in/' | relative_url }}) |
| Make a silo and step inside | [Your first silo]({{ '/use/silos/first-silo/' | relative_url }}) |
| Pick a microVM, a container or a sandbox | [Runtimes]({{ '/use/silos/runtimes/' | relative_url }}) |
| Run many copies, update an image | [Sets, images and updates]({{ '/use/silos/sets/' | relative_url }}) |
| Send traffic only to silos that are ready | [Probes and routing]({{ '/use/silos/probes/' | relative_url }}) |
| Give silos volumes, or make a microVM movable | [Volumes]({{ '/use/silos/volumes/' | relative_url }}), [Moves]({{ '/use/silos/moves/' | relative_url }}) |
| Keep a program running inside | [Daemons and upkeep]({{ '/use/silos/daemons/' | relative_url }}) |
| Get a shell, copy files | [Terminals, files and the small UI]({{ '/use/silos/terminals/' | relative_url }}) |
| Show a status to the people watching | [Panes]({{ '/use/panes/' | relative_url }}) |
| Understand addresses, ports and names | [Addresses, ports and names]({{ '/use/networking/model/' | relative_url }}) |
| Open a port to another silo, or let a silo reach outside | [Networking]({{ '/use/networking/' | relative_url }}) |
| Keep a port closed until you need it | [Port leases]({{ '/use/networking/leases/' | relative_url }}) |
| Serve a site over HTTPS | [HTTPS sites]({{ '/use/ingress/' | relative_url }}) |
| Give a silo a secret without installing anything in it | [Secrets]({{ '/use/secrets/' | relative_url }}) |
| Add commands to `kvo` | [kvo plugins]({{ '/use/kvo-plugins/' | relative_url }}) |
| Look up a command | [kvo commands]({{ '/reference/kvo/' | relative_url }}) |

---

## What an admin does for you

Namespaces, roles, CI tokens, opening ports to the outside, addresses you keep, host names on an
ingress class, and reading a secret's value back are an admin's. Each page says where it needs one,
and [What an admin does for you first]({{ '/use/sign-in/#what-an-admin-does-for-you-first' | relative_url }})
lists them in one place.
