---
title: Addresses you keep
permalink: /admin/ips/
redirect_from: /networking/ips/
---

# Addresses you keep

Some addresses must outlive the silo using them: a peer allow-lists it, a DNS record points at it.
An **IP** is an address kept as an object of its own, so it stays when a silo or service goes away.

The words used here (internal IP, external IP, silo IP, service, port) are defined on
[Addresses, ports and names]({{ '/use/networking/model/' | relative_url }}). This page covers the
`loadBalancer` part of a service port: where its address comes from.

---

## Where a load-balancer address comes from

A port with `loadBalancer: true` is reachable on a **load-balancer address**: an address in front of
the host, from a class. The silo names the class:

```yaml
kind: Silo
metadata: { name: api-blue, namespace: payments }
spec:
  runtime: cloud-hypervisor
  vm: { image: ubuntu-24.04 }
  ports:
    - { name: https, port: 8443 }
  service:
    name: api
    ports:
      - { port: https, as: 443, loadBalancer: true }
    loadBalancer: { class: ip-pool }
```

| `service.loadBalancer` | Means |
|---|---|
| `class` | Which class provides the address. Required |
| `ip` | The name of an IP object: this address and no other |
| `dedicated: true` | An address of its own, made for this service and named after it, instead of the shared one |

Which classes exist:

| Class | Address | Who provides it |
|---|---|---|
| `ip-pool` | One of a host's extra addresses | Built in. See [Extra addresses on a host](#extra-addresses-on-a-host) |
| Others | A load balancer or an address a provider owns | A [provider plugin]({{ '/admin/extending/plugins/' | relative_url }}), such as the ones on the provider pages. Each provider's page lists its classes |

Without `ip` and without `dedicated`, **services share**:

- Every load-balancer service of a class in a namespace that names no address shares one IP, named
  `default-CLASS`, each on its own ports. Kevaso makes it on first use, and it goes with its last
  service unless you [keep it](#keep-and-release).
- Two services can't both take the same port number on it: the second is refused, and the refusal
  names who has it.
- **A namespace never shares an address with another namespace.**
- The instances of a SiloSet have the same ports, so each has an address of its own.
- An address that only one host can have at a time (every `ip-pool` address, and an announced shared
  address a plugin says is exclusive) is never shared: one silo holds it.

`ip-pool` addresses are for one silo's own service. A standalone `Service` of `type: LoadBalancer`
can't use `ip-pool`.

A standalone `Service` takes the same choices as fields: `loadBalancerClass`, `loadBalancerIP` and
`loadBalancerDedicated` (not together with `loadBalancerIP`).

---

## Make one by hand

```bash
kvo create ip payments-edge -n payments --class ip-pool --host baremetal-1
```

```text
ip/payments-edge created: its address shows in kvo get ips once reserved
```

| Flag | Meaning |
|---|---|
| `--class CLASS` | Required: `ip-pool`, or a provider's class |
| `--host HOST` | `ip-pool`: the host whose extra addresses it picks from. Without it, the first host with one free |
| `--region REGION` | A provider's class: the region to reserve in. Without it, where its first holder lands |
| `--address A` | Ask for this address: one of the host's, or one you already own at the provider. Without it, any |
| `--reclaim Delete` | Delete the IP when nothing holds it. The default for one you make is `Retain` |

Name it from the silo (naming an IP needs the verb `use` on it, like a secret; editors and admins
have it):

```yaml
service:
  loadBalancer: { class: ip-pool, ip: payments-edge }
```

An IP's class, and where it is once it has an address, never change. Deleting an IP something
holds is refused.

```bash
kvo get ips -n payments
```

```text
NAME             ADDRESS        CLASS    HELD-BY             KEPT
payments-edge    203.0.113.21   ip-pool  silo/api-blue (443/TCP)  yes
default-lb-1     198.51.100.7   lb-1     service/web (80/TCP)     no, goes with its last service
```

| Column | Meaning |
|---|---|
| `ADDRESS` | The address, once reserved |
| `CLASS` | Where it comes from |
| `HELD-BY` | The silos and services on it, with their ports |
| `KEPT` | `yes`, or `no, goes with` the object it was made for |

```bash
kvo describe ip payments-edge -n payments
```

`describe` shows the address, its class (and the host or region), whether it follows its holder to
another host, whether it is kept, who holds it, any service waiting for it and why, and for each
silo holding it the line of what comes in and out (below).

---

## Keep and release

An IP Kevaso made for a service goes with it. Keep it, and it stays until you delete it:

```bash
kvo ip keep api-blue -n payments
```

```text
ip/api-blue kept: 203.0.113.22 stays until you delete the IP
```

To let it go again:

```bash
kvo ip release api-blue -n payments
```

If nothing holds it, it is deleted at once and the address goes back. Otherwise it goes when its
holder lets go of it.

---

## Extra addresses on a host

The `ip-pool` class hands out a host's own extra addresses, ones routed to that machine. You tell
Kevaso which:

```bash
kvoadm host addresses baremetal-1 add 203.0.113.20/30
```

```text
→ baremetal-1's extra addresses: 203.0.113.20/30
```

```bash
kvoadm host addresses baremetal-1 list
```

```text
ADDRESS         IP
203.0.113.20    -
203.0.113.21    ip payments/payments-edge (held by silo/api-blue (443/TCP))
```

```bash
kvoadm host addresses baremetal-1 remove 203.0.113.20/30
```

- A range can't be removed while an IP holds an address in it. An address can't be routed to two
  hosts.
- A silo using one runs only on that host. If no host has one free, `kvo describe silo` says
  `kvoadm host addresses HOST add CIDR`.

---

## In and out

A silo's outgoing traffic leaves from the host's address unless you choose otherwise.
`network.egress.source` picks:

| `egress.source` | Outgoing traffic leaves from |
|---|---|
| `host` (default) | The host's address |
| `sameAsService` | The address the silo is reached on |
| `pool` | An `ip-pool` IP of its own, named in `egress.ip` (its service's, when that is `ip-pool`) |

```yaml
spec:
  network:
    egress: { source: sameAsService }
```

Whether that can be honoured depends on the class:

| The service is reached on | `sameAsService` sends from |
|---|---|
| A host port | The host's address |
| An `ip-pool` address | That address |
| A class whose address sits on the silo's host (an announced shared address) | That address |
| A balancer in front of the hosts | Nothing. The balancer's address is on no host, so nothing can send from it. Refused when a plugin serves the class; if none does now, the silo sends from the host's address |

When the address out isn't the address in, `kvo describe silo` says why:

```text
same address in and out: 203.0.113.21
```

Other lines are `in 203.0.113.21, out through the host's NAT`, or an explanation such as "waiting for IP
NAME's address: out is the host's meanwhile".

---

## What isn't done

- An external IP that is not on any of the host's interfaces (a cloud's 1:1 NAT address) matches
  nothing: `hostPort: [external]` doesn't work there yet.
