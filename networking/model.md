---
title: Addresses, ports and names
permalink: /networking/model/
---

# Addresses, ports and names

This page is the reference for how things are addressed in Kevaso: what address a host and a silo
have, how a port becomes reachable, and what names resolve to. Every other page uses these words the
way they are defined here.

> **Status.** This is the model Kevaso is being moved to. Parts of it are in the code today and parts
> are not: see [what's built](#whats-built) at the end. The rest of the docs still use the older
> forms where the code does.

---

## The five addresses

![Two hosts with their external, internal and mesh IPs, and silos with silo IPs]({{ '/assets/addresses.svg' | relative_url }})

| Term | Belongs to | Who sets it | Meaning |
|---|---|---|---|
| **Internal IP** | A host | You, when you add the machine | Its address on its own network |
| **External IP** | A host | You, optional | Its address reachable from outside that network |
| **Mesh IP** | A host | Kevaso | Its address on the encrypted network between hosts |
| **Silo IP** | A silo | Kevaso | The silo's own address inside the cluster. It changes when the silo changes host |
| **Cluster IP** | A `Service` | Kevaso | One stable address in front of several silos. Never one silo's address |

A host has at least one of internal and external:

```yaml
kind: Host
metadata: { name: ovh-1 }
spec:
  internalIP: 10.0.0.5
  externalIP: 51.38.10.20
```

| Machine | Internal IP | External IP |
|---|---|---|
| A cloud VM with a private and a public address | Yes | Yes |
| A bare-metal server with one address on the internet | No | Yes |
| A machine on a LAN, or behind NAT | Yes | No |

All three are ordinary hosts. Kevaso has no word for "public", "internet" or "LAN": it can't know
which is which, so you say.

![A Linode, two bare-metal hosts and three hosts on a LAN, all on one mesh]({{ '/assets/addresses-mixed-cluster.svg' | relative_url }})

---

## Ports: declare, then open

A silo **declares** its ports. A **service** on the silo opens some of them.

```yaml
kind: Silo
metadata: { name: api-blue, namespace: payments }
spec:
  ports:
    - { name: http,    port: 8080 }
    - { name: https,   port: 8443 }
    - { name: metrics, port: 9090 }
    - { name: debug,   port: 9229 }

  service:
    name: api
    ports:
      - { port: http,    allowFrom: [nginx-public, billing/worker] }
      - { port: https,   hostPort: [external], as: 443 }
      - { port: metrics, hostPort: [internal], sources: ["10.0.0.0/24"] }
      - { port: debug,   hostPort: [internal], lease: { max: 2h } }
```

**`spec.ports`** says what exists.

- A port that isn't declared can never be reached, by anything. A service that names one is refused.
- The program inside may listen on other ports. Nothing can connect to them.

**`spec.service`** says what is reachable.

| Field | On | Meaning |
|---|---|---|
| `name` | The service | The name others use. Leave it out and it's the silo's name |
| `port` | A service port | Which declared port this is about |
| `allowFrom` | A service port | The silos that may connect: `NAME`, or `NAMESPACE/NAME` |
| `hostPort` | A service port | Which of the host's addresses it opens on: `[internal]`, `[external]`, or both. Absent: not on the host |
| `as` | A service port | The number on the host, when it differs from the silo's |
| `sources` | A service port | The network ranges that may connect on the host. Absent: anyone who can reach that address |
| `lease` | A service port | The port opens only for a time, on request. See [Port leases]({{ '/networking/leases/' | relative_url }}) |

**A silo with no service** has a silo IP for its own outgoing traffic, no name, and nothing anyone
can connect to. That is a normal silo: a worker that only calls out needs nothing more.

**Where a silo can run** follows from its ports. A port with `hostPort: [external]` needs a host that
has an external IP, and a host where no other silo already holds that address and number.
`kvo describe silo` says which rule excluded each host.

---

## Three ways in

![One API behind three ingress setups: hosts with their own external IP, a LAN behind one shared external IP, and internal only]({{ '/assets/three-ways-in.svg' | relative_url }})

| Case | The port asks for | In front | Who can reach it |
|---|---|---|---|
| Its own external IP | `hostPort: [external]` | Nothing | Anyone |
| One shared external IP | `hostPort: [internal]` | A balancer that holds the external IP | Anyone, through the balancer |
| Internal only | `hostPort: [internal]` | Nothing | Machines on that network |

The second and third are written the same way. What sits in front is your network's business:
Kevaso opens the port on the host and stops there.

---

## Names

A name belongs to a service or to a host. Nothing else has one.

```text
SERVICE . NAMESPACE [ . NETWORK ] . kevaso
HOST    . NETWORK                 . kevaso
```

`NETWORK` is `mesh`, `internal` or `external`. Those three words can't be namespace names.

For the service above, on host `ovh-1`:

| Name | Answer |
|---|---|
| `api.payments.mesh.kevaso` | `10.64.2.7`, the silo IP |
| `api.payments.internal.kevaso` | `10.0.0.5`, because `metrics` and `debug` have `hostPort: [internal]` |
| `api.payments.external.kevaso` | `51.38.10.20`, because `https` has `hostPort: [external]` |
| `api.payments.kevaso` | The best of the three for whoever asks |
| `ovh-1.mesh.kevaso` | `10.63.0.2` |
| `ovh-1.internal.kevaso` | `10.0.0.5` |
| `ovh-1.external.kevaso` | `51.38.10.20` |

**The plain name is dynamic.** Leave the network out and Kevaso answers with the address that works
from where you are:

| Who asks for `api.payments.kevaso` | Gets |
|---|---|
| A silo | The silo IP |
| A machine on the host's network | The host's internal IP |
| Anyone else | The host's external IP |

If the service isn't reachable from there, there is no answer.

**Short names** work inside a silo: `api` in the same namespace, `api.payments` from another.

**A service name has one owner** in its namespace. Two silos can't use the same one. Several silos
behind one name is a `Service` or a `SiloSet`, which has a cluster IP.

### Who can resolve what

Having a name doesn't mean everyone can look it up.

| Who asks | Can resolve |
|---|---|
| A silo | Services, by their plain name. No host names, no internal or external answers |
| A host in the cluster, an operator | Everything |
| A machine on a host's network | Internal and external names |
| Anyone else | External names |

Two more rules:

- **Resolving a name isn't permission.** `allowFrom` decides whether a silo's connection is accepted.
- **`.kevaso` names are answered by Kevaso's own DNS**, which runs on its hosts. A machine outside
  the cluster resolves them only if it asks a Kevaso host. Your public hostnames stay in your own
  DNS, pointing at a host's external IP or at whatever you put in front.

---

## What's built

| Part of the model | In the code today |
|---|---|
| Silo IP, mesh IP, cluster IP for a `Service` | Yes |
| `NAME.NAMESPACE.kevaso` for a silo, and short names | Yes, for every silo, whether or not it has a service |
| Silo-to-silo access by `allowFrom` | Yes, as a list on the silo |
| Ports opened on the host | Yes, as `expose: hostPort` for the whole service, on every address the host has |
| Internal and external IP on a host | No. A host has one address |
| Ports declared on the silo, opened by the service | No. Ports are listed inside `service` |
| `hostPort: [internal]` / `[external]` per port, `as`, `sources` | No |
| A service name different from the silo's | No |
| The `mesh`, `internal` and `external` names, host names, and the dynamic answer | No |
| Limits on who can resolve what | No. A silo can resolve any silo's name |
