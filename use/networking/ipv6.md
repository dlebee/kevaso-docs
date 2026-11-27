---
title: IPv6 at the edge
permalink: /use/networking/ipv6/
---

# IPv6 at the edge

Kevaso is **dual stack at the edge**: a silo can answer on IPv6, and send to IPv6 destinations,
where its worker has a global IPv6 address. Inside the cluster everything stays IPv4: the mesh,
`Internal` services, `.kevaso` names and the addresses they resolve to.

IPv6 is added beside IPv4, never instead of it. The words used here (external IP, silo IP,
service) are defined on [Addresses, ports and names]({{ '/use/networking/model/' | relative_url }}).

---

## What answers on IPv6

| Way in | On IPv6 |
|---|---|
| `HostPort`, `worker: external` (the default) | **Yes**, on the worker's own IPv6 address (`externalIPv6`), when it has one. The same ports as on its IPv4 address |
| `HostPort`, `worker: internal` | No: the internal IP is IPv4 |
| `HostPort`, `ip: NAME`, an IP of `family: IPv6` | **Yes**, on that address. An IP of `family: IPv4` answers on IPv4 only |
| `HostPort`, `own: true` | As the instance's member: an IP set is [one family](#one-ip-is-one-family) |
| `LoadBalancer` | Where the provider gives its balancers an IPv6 address, shown as `loadBalancerIPv6` in the status and in `kvo describe`. On Linode, NodeBalancers answer on IPv6 ([Linode]({{ '/admin/providers/linode/#a-nodebalancer-in-front' | relative_url }})) |
| `Internal`, and a silo reaching another silo | No: the mesh is IPv4 |

An IPv6 connection is forwarded to the silo's own IPv6 address, as an IPv4 one is to its silo IP.
It is plain forwarding, so **the silo sees the client's IPv6 address**.

Every silo on a worker with IPv6 gets an IPv6 address of its own, beside its silo IP. It is
private to that worker, like the silo IP: it isn't reachable from outside or from another host,
and it changes when the silo changes host. `kvo describe silo` shows it as `Silo IPv6`:

```bash
kvo describe silo hello -n demo
```

```text
Silo IP:                10.64.2.7
Silo IPv6:              fd12:3456:789a:2::7
Egress IP:              -
Egress IPv6:            2001:db8:10::5
```

On a worker without IPv6, it says why there is none:

```text
Silo IPv6:              none: host baremetal-1 has no IPv6 subnet (IPv4 only)
Egress IPv6:            none (no IPv6 out on its worker)
```

---

## A host without IPv6 is IPv4 only

A worker has IPv6 when it has a **global IPv6 address**: Kevaso finds the one it uses to reach the
IPv6 internet, and keeps it as the host's `externalIPv6`. A host without one is IPv4 only. That is
not an error: its silos still run, and answer on IPv4.

```bash
kvo describe host baremetal-1
```

```text
External IP:        203.0.113.20
External IPv6:      none: IPv4 only (no global IPv6 address; set spec.externalIPv6 if it has one)
```

A silo that asks for `worker: external` is placed on workers with or without IPv6: IPv6 is given
where the worker has it, and a worker without it is not refused. A service on a kept IPv6 address
runs where that address is, as for IPv4 ([below](#a-kept-address-on-both-families)).

What a machine needs for IPv6 is on [Prepare a machine]({{ '/admin/setup/prepare/#ipv6' | relative_url }}).

---

## One IP is one family

An [IP]({{ '/use/networking/ips/' | relative_url }}) you keep is exactly one address, so it is
either IPv4 or IPv6. `family` says which, and is `IPv4` when you leave it out:

```yaml
apiVersion: kevaso/v1
kind: IP
metadata: { name: edge6, namespace: payments }
spec:
  type: Host
  family: IPv6
  hostAddress: edge6        # a host's extra address that is IPv6
```

- A `Host` IP's family is its host address's. Naming an IPv6 address from an IP without
  `family: IPv6` is refused, with the fix: `set family: IPv6, or name an IPv4 address`.
- `family` can't change: make another IP.
- `kvo create ip` makes IPv4 IPs. Make an IPv6 one with YAML and `kvo apply -f`.
- An IP set is one family too: see [IP sets]({{ '/use/networking/ips/#one-family-per-set' | relative_url }}).

Where IPv6 addresses to keep come from depends on where the hosts are:

| Where | IPv6 addresses you can keep |
|---|---|
| Bare metal, or any host with IPv6 routed to it | Its extra addresses, added by an administrator ([Host addresses and providers]({{ '/admin/addresses/' | relative_url }})) |
| Linode | Extra addresses from a `/64` an administrator adds to that Linode. A floating (`type: Provider`) IP on Linode is IPv4 only, and an IPv6 one is refused when you apply ([Linode]({{ '/admin/providers/linode/#ipv6' | relative_url }})) |
| A provider that declares `providerIPv6` | Floating IPv6 IPs (`type: Provider`, `family: IPv6`), one address each, following the silo like IPv4 ones |

---

## A kept address on both families

One service has one way in, and one IP is one family. A service that should answer on a kept
address over IPv4 and IPv6 is therefore **two services**, one on each IP. A port can be in several
services, so both open the same port:

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: api, namespace: payments }
spec:
  runtime: docker
  container: { image: ghcr.io/acme/api:v12 }
  ports:
    - { name: https, port: 8443 }
  services:
    - name: api
      type: HostPort
      hostPort: { ip: edge }       # family IPv4 (the default)
      ports: [{ port: https, as: 443 }]
    - name: api6
      type: HostPort
      hostPort: { ip: edge6 }      # family: IPv6
      ports: [{ port: https, as: 443 }]
```

- The silo runs where both addresses can be. Two `Host` IPs must be on the same host, or the silo
  can never run, and it is refused.
- A SiloSet names one IP set, of one family, so its instances keep addresses of one family. A
  service on `worker: external` answers on both of the worker's addresses.

---

## A public service still needs IPv4

Many clients have no IPv6. A service on IPv6 only is reachable from fewer places than you might
expect, so give a public service an IPv4 address too: the worker's (`worker: external` answers on
both), or a second service on an IPv4 IP, as above.

---

## Sending to IPv6

Outgoing traffic goes out in the destination's family:

- To an IPv6 destination, a silo sends from its **worker's IPv6 address**, or from a kept IPv6 IP
  with `egress.source: { ip: NAME }`.
- `egress.source` sets the source for **its IP's family only**. A silo sending from a kept IPv4 IP
  still sends to IPv6 destinations from its worker's IPv6 address, unless it names a kept IPv6 IP
  instead. `kvo describe silo` shows both as `Egress IP` and `Egress IPv6`.
- On a worker without IPv6, an IPv6 destination is unreachable at once. Nothing leaves over IPv4
  instead.

As for IPv4, **nothing leaves without a Grant**. A Grant takes IPv6 addresses and ranges:

```yaml
apiVersion: kevaso/v1
kind: Grant
metadata: { name: partner, namespace: payments }
spec:
  egress:
    - { to: "203.0.113.0/24", ports: ["443"] }
    - { to: "2001:db8:44::/48", ports: ["443"] }
```

A host-name rule (`to: api.example.com`) connects over IPv4: write the IPv6 address or range to
reach a destination over IPv6. What a grant never opens is on
[Networking]({{ '/use/networking/#what-a-silo-may-reach-outside' | relative_url }}).

---

## What stays IPv4

- The mesh, `Internal` services, `allowFrom`, cluster IPs and `.kevaso` names.
- A host's internal IP and its external IP (IPv6 goes in `externalIPv6`), so `worker: internal`.
- `sources` on a service port take IPv4 only. A port with `sources` is closed on IPv6: no IPv6
  client is in its list.
- A host-name rule in a Grant.
- The API (`6443`).
- A [Mac worker]({{ '/admin/workers/mac/' | relative_url }}).
