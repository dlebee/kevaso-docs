---
title: Networking
permalink: /networking/
---

# Networking

Kevaso owns the firewall on its hosts, and starts from **deny**. This page goes through every way
traffic is opened, one rule at a time.

---

## The words used on this page

Internal IP, external IP, mesh IP, silo IP and cluster IP each mean one thing in Kevaso, and so do
"port", "service" and a `.kevaso` name. They're defined, with pictures, in
[Addresses, ports and names]({{ '/networking/model/' | relative_url }}). This page doesn't repeat them.

---

## The defaults

| From → to | Default | Opened by |
|---|---|---|
| A silo → another silo | Denied | The receiving port's `allowFrom` |
| Outside → a silo | Denied | A service port with `hostPort` or `loadBalancer: true` |
| A silo → outside | Denied | A `Grant` naming the destination ([below](#what-a-silo-may-reach-outside)) |
| A silo → its host | DNS and ping only | Never more, except the [local endpoint]({{ '/secrets/' | relative_url }}) for a silo that has something to ask it, and the egress gate for a silo whose grants name hosts |
| A silo → the mesh's host addresses | Denied | Never |
| A silo → names (DNS) | Allowed | Its host answers on the silo network's gateway |

Each silo also can't send packets with an address other than its own (anti-spoofing), so a rule
"from silo X" really means silo X.

---

## Declare a port, then open it

A silo declares its ports. Its service opens them, one at a time, in one of three ways. Everything
about one port is in one entry.

```yaml
spec:
  ports:
    - { name: http,  port: 8080 }
    - { name: stats, port: 9125, protocol: UDP }
    - { name: admin, port: 9090 }
  service:
    ports:
      - { port: http,  hostPort: [external], as: 80 }
      - { port: stats, allowFrom: [metrics-collector] }
      - { port: admin, hostPort: [internal], sources: ["10.0.0.0/24"] }
```

A port the service doesn't list is open to nobody. A silo with no `service` has no name and nothing
reachable: it can still call out.

### Letting another silo in: `allowFrom`

`allowFrom` on a service port names the silos that may connect to it on its silo IP: `NAME` for the
same namespace, `NAMESPACE/NAME` for another. It works across hosts too, over the mesh. A silo named
here that isn't running opens nothing. `allowFrom` takes silos only: a network range there is
refused, and ranges go in `sources`.

### On the host's address: `hostPort`

`hostPort` opens the port on the host's `internal` address, its `external` address, or both. `as` is
the number there when it differs from the silo's. `sources` limits who may connect to network ranges.

It is plain layer-4 forwarding: no proxy, and **the silo sees the client's real address**. The host
never answers a host port on Kevaso's own networks (the mesh, the silos, the Services' virtual IPs).

A port with `hostPort: [external]` needs a host with an external IP. See
[where a silo can run]({{ '/networking/model/' | relative_url }}#ports-declare-then-open).

### On an address Kevaso manages: `loadBalancer`

A port with `loadBalancer: true` is reachable on the service's load-balancer address, at its `as`
number. The `service.loadBalancer` block says where the address comes from.

| Class | Address | Needs |
|---|---|---|
| `ip-pool` | An extra address routed to the silo's host | Extra addresses declared on that host |
| A class from a provider plugin | A balancer's address, or an IPv4 that follows the silo | A [provider plugin]({{ '/extending/plugins/' | relative_url }}); each provider's page lists its classes |

```yaml
spec:
  ports:
    - { name: web, port: 8080 }
  service:
    ports:
      - { port: web, loadBalancer: true, as: 80, sources: ["203.0.113.0/24"] }
    loadBalancer: { class: ip-pool }
```

Which address, whether it is shared, and keeping it when the silo goes are on
[Addresses you keep]({{ '/networking/ips/' | relative_url }}). A class that can't limit clients by address
can't serve a port that has `sources`: see [leases and sources]({{ '/networking/leases/' | relative_url }}#load-balancers-from-a-plugin-and-sources).

### Only for a while: a lease

Some ports you want closed until you need them: a debug port, an admin page. Put a `lease` on the
port and it starts closed, opens for a time you choose, and closes by itself:
[Port leases]({{ '/networking/leases/' | relative_url }}).

### Plain HTTP and HTTPS sites

For HTTP and HTTPS by host name and path, with certificates, use an [ingress]({{ '/ingress/' | relative_url }}).
The silos behind it stay private.

> **Opening a port to the outside is its own permission:** `create` on `services/expose-public` in
> the namespace. It is needed for a silo with `hostPort` or `loadBalancer: true` ports, for a SiloSet
> whose instances would have them, and for a standalone Service of type `LoadBalancer`
> ([Sets, images and updates]({{ '/silos/sets/' | relative_url }})). Namespace `admin`s have it,
> `editor`s don't.

---

## Which address its traffic leaves from

```yaml
spec:
  network:
    egress:
      source: sameAsService     # host (default) | sameAsService | pool
```

| `source` | Outgoing traffic leaves from |
|---|---|
| `host` (default) | The host's own address. A host behind NAT has none to name |
| `sameAsService` | The address the silo is reached on |
| `pool` | An extra address of class `ip-pool`, named by `egress.ip` (or the one the silo's `ip-pool` load balancer uses) |

If a silo tells others the address it's reached on and they must see its outbound traffic come from
that same address, set `sameAsService`. It needs a service with a port on the host or a load balancer,
and Kevaso refuses it otherwise. Whether the address can be sent from depends on how the port is
opened. A fourth source, `gateway` (leave from a chosen host), is not built.

---

## Can a silo use the same IP out as in?

| The port is open on | The address people reach it on | Same address for outgoing traffic? |
|---|---|---|
| Nothing (no service) | None | Nothing to match. `sameAsService` is refused |
| `hostPort` | The host's address | **Yes.** `host` and `sameAsService` are the same address, so the default is already that |
| A load balancer of class `ip-pool` | An address of its own | **Yes**, with `sameAsService` (or `pool`). That address is on the host, so the host sends from it |
| A provider class whose address is announced to the silo's host (a shared IP) | The shared IP | **Yes**, with `sameAsService`. The host holds the address while the silo is there |
| A provider class that puts a balancer in front | The balancer's address | **No.** That address is on no host, so nothing can send from it. `sameAsService` is refused |
| An [Ingress]({{ '/ingress/' | relative_url }}) controller | The controller's address | **No.** The address belongs to the controller, not to your silo |

`sameAsService` is refused for a class whose address is a balancer's, on no host, when a provider plugin
serves that class. When Kevaso can't tell yet (no plugin serves the class now), or the address is still
being made, the silo sends from the host's address and `kvo describe silo` says why.

To see both addresses for a running silo:

```bash
kvo describe silo web -n shop
```

```text
Egress IP:   172.105.3.63 (sameAsService)
In and out:  same address in and out: 172.105.3.63
```

`Egress IP` is where its outgoing traffic leaves from, with the `source` you chose in brackets. `In
and out` says in one line whether the address out is the address in, and why not when it isn't. A `-`
means there is none, for example on a host behind NAT.

---

## What a silo may reach outside

Nothing, until a **Grant** says so. A Grant is its own object; a silo lists the ones it uses:

```yaml
apiVersion: kevaso/v1
kind: Grant
metadata: { name: build-tools, namespace: dev }
spec:
  egress:
    - { to: api.example.com }                                  # a name: TLS on 443
    - { to: "*.example.org" }                                  # a wildcard name
    - { to: "203.0.113.0/24", ports: ["5432"] }                # raw, an address range, one port
---
apiVersion: kevaso/v1
kind: Silo
metadata: { name: builder, namespace: dev }
spec:
  runtime: cloud-hypervisor
  vm: { image: ubuntu-24.04 }
  grants: [build-tools]
```

| `to:` | How it's enforced |
|---|---|
| An address or CIDR | The host firewall, by address, port and protocol |
| A host name | The host's **egress gate**: the silo's TLS is checked against the name it asks for (SNI) and passed through, untouched, only for that name, to the address the gate looks up itself. No name or another name: refused. A name opens TCP only, port 443 unless you list ports. (The gate doesn't look inside a passed-through connection; for hosts it injects a secret for, it does, and refuses a request for another host: see [Secrets]({{ '/secrets/' | relative_url }}).) |

A rule opens TCP unless it lists `protocols` (`tcp`, `udp`; a host name is TCP only). `ports` takes
numbers and ranges such as `"30000-30100"`; with none, an address range opens every port and a host name
opens 443. Addresses are IPv4 only for now.

DNS always works, so a silo can resolve anything; it just can't connect where no grant says.
