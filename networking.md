---
title: Networking
permalink: /networking/
---

# Networking

Kevaso owns the firewall on its hosts, and starts from **deny**. This page goes through every way
traffic is opened, one rule at a time.

---

## The defaults

| From → to | Default | Opened by |
|---|---|---|
| A silo → another silo | Denied | The receiving silo's `allowFrom` |
| The internet → a silo | Denied | The silo's `service` with `expose: hostPort` or `loadBalancer` |
| A silo → the internet | Denied | A `Grant` naming the destination ([below](#what-a-silo-may-reach-outside)) |
| A silo → its host | DNS and ping only | Never more |
| A silo → the mesh's host addresses | Denied | Never |
| A silo → names (DNS) | Allowed | Its host answers on the silo network's gateway |

Each silo also can't send packets with an address other than its own (anti-spoofing), so a rule
"from silo X" really means silo X.

---

## Private names

Every silo with a service gets a name inside the cluster:

```
NAME.NAMESPACE.kevaso      api.payments.kevaso
NAME                       api            (from a silo in the same namespace)
```

Each worker's DNS answers them for its silos, and forwards everything else.

---

## A service of its own

```yaml
spec:
  service:
    ports:
      - { name: http, port: 8080 }
      - { name: p2p,  port: 30303, protocol: UDP }
```

That alone is **private**: reachable by name, but only by silos you allow.

---

## Letting another silo in: `allowFrom`

On the silo being reached:

```yaml
spec:
  allowFrom:
    - { silo: worker, ports: [http] }     # same namespace, only the http port
    - { silo: research/crawler }          # another namespace, every port
```

It works across hosts too (over the mesh). A silo named here that isn't running opens nothing.

---

## Open to the internet

### On the host's own address: `hostPort`

```yaml
spec:
  service:
    ports:
      - { name: p2p,  port: 30303, protocol: TCP }
      - { name: disc, port: 30303, protocol: UDP }
      - { name: rpc,  port: 8545 }
    expose: hostPort
    hostPort:
      ports: [p2p, disc]              # only these; rpc stays private
      allowFrom: ["203.0.113.0/24"]   # optional: only these sources
```

Plain layer-4 forwarding: no proxy, and **the silo sees the client's real address**.

### Its own address: `loadBalancer`

| Class | Address | Needs |
|---|---|---|
| `worker-ip` | The host's own address (like `hostPort`, chosen for you) | Nothing |
| `ip-pool` | An address of its own, from an `IPPool` routed to the host | Extra addresses on that host |
| `linode-nodebalancer`, `linode-shared-ip` | A NodeBalancer's, or an IPv4 that follows the silo | The [Linode provider]({{ '/providers/linode/' | relative_url }}) |

```yaml
apiVersion: kevaso/v1
kind: IPPool
metadata: { name: hv2-extra }
spec: { host: hv2, addresses: ["198.51.100.17/32", "198.51.100.18/32"] }
---
# in the silo
spec:
  service:
    ports: [{ name: p2p, port: 30303 }]
    expose: loadBalancer
    loadBalancer: { class: ip-pool, pool: hv2-extra, stableIP: true }
```

### Only for a while: a lease

Some ports you want closed until you need them: a debug port, an admin page. Put a `lease` on the
service and the port starts closed, opens for a time you choose, and closes by itself:
[Port leases]({{ '/networking/leases/' | relative_url }}).

> **Exposing publicly is its own permission:** `create` on `services/expose-public`. Namespace
> `admin`s have it, `editor`s don't. The same goes for a SiloSet whose instances would be exposed,
> and a standalone Service of type `LoadBalancer` ([Sets, images and updates](../silos/sets/)).

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
| `host` (default) | The host's own address |
| `sameAsService` | The address the silo is reached on (its `Public IP`) |
| `pool` | An address from an `IPPool`; also set `egress: { source: pool, pool: NAME }` |

`sameAsService` is what P2P nodes need: they advertise the address they're reached on, so their
outbound traffic must come from it too. It needs a service exposed with `hostPort` or
`loadBalancer`. Whether it can work depends on how the silo is exposed:
[next section](#can-a-silo-use-the-same-ip-out-as-in). A fourth source, `gateway` (leave from a
chosen host), is not built.

---

## Can a silo use the same IP out as in?

Only some ways of exposing a silo let it. This is what the code does today:

| Exposed with | The address people reach it on | Same address for outgoing traffic? |
|---|---|---|
| Private (no `expose`) | None | Nothing to match. Outgoing uses `host` (default) or `pool`. `sameAsService` is refused. |
| `hostPort` | The host's address | **Yes.** `host` and `sameAsService` are the same address here, so the default is already that. |
| `loadBalancer`, class `worker-ip` | The host's address | **Yes**, same as `hostPort`. |
| `loadBalancer`, class `ip-pool` | An address of its own from the pool | **Yes**, with `sameAsService` (or `pool` with the same pool). That address is on the host, so the host sends from it. |
| `loadBalancer`, a provider's announced address (`linode-shared-ip`) | The shared IP | **Yes**, with `sameAsService`. The silo's host holds the address while the silo is there, and the address moves with it. |
| `loadBalancer`, a provider's balancer in front (`linode-nodebalancer`) | The balancer's IP | **No.** That address is on no host, so nothing can send from it. Use `host` or `pool`. |
| Behind an [Ingress]({{ '/ingress/' | relative_url }}) | The ingress controller's address | **No.** That address belongs to the controller, not to your silo. The silo is private, so use `host` or `pool`. |

**Not refused yet:** `sameAsService` on a silo behind a balancer (`linode-nodebalancer`) is accepted
when you apply it, but it can't work: outgoing connections would be sent from an address the host
doesn't hold. Don't set it. Kevaso will refuse it in a later change.

To see both addresses for a running silo:

```bash
kvo describe silo web -n shop
```

```text
Exposure:    loadBalancer (linode-shared-ip)
Public IP:   172.105.3.63
Egress IP:   172.105.3.63 (sameAsService)
```

`Public IP` is where people reach it. `Egress IP` is where its outgoing traffic leaves from, with the
`source` you chose in brackets. With the default (`host`) the bracket is absent and the address is
the host's. A `-` means there is none (for example, a host that is behind NAT). If the silo is
announced, `describe` also prints an `Announced` line.

---

## What a silo may reach outside

Nothing, until a **Grant** says so. A Grant is its own object; a silo lists the ones it uses:

```yaml
apiVersion: kevaso/v1
kind: Grant
metadata: { name: build-tools, namespace: dev }
spec:
  egress:
    - { to: api.github.com }                                  # a name: TLS on 443
    - { to: "*.npmjs.org" }                                   # a wildcard name
    - { to: "0.0.0.0/0", ports: ["30303"], protocols: [tcp, udp] }   # raw, any address, one port
---
apiVersion: kevaso/v1
kind: Silo
metadata: { name: builder, namespace: dev }
spec:
  runtime: cloud-hypervisor
  image: ubuntu-24.04
  grants: [build-tools]
```

| `to:` | How it's enforced |
|---|---|
| An address or CIDR | The host firewall, by address, port and protocol |
| A host name | The host's **egress gate**: the silo's TLS is checked against the name it asks for (SNI) and passed through, untouched, only for that name, to the address the gate looks up itself. No name or another name: refused. (The gate doesn't look inside a passed-through connection; for hosts it injects a secret for, it does, and refuses a request for another host: see [Secrets]({{ '/secrets/' | relative_url }}).) |

DNS always works, so a silo can resolve anything; it just can't connect where no grant says.
