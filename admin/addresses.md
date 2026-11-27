---
title: Host addresses and providers
permalink: /admin/addresses/
redirect_from:
  - /admin/ips/
  - /networking/ips/
---

# Host addresses and providers

Teams keep addresses as [IPs and IP sets]({{ '/use/networking/ips/' | relative_url }}) in their
namespaces. Where those addresses come from is set up here, for the whole cluster:

- a host's **named extra addresses**, which an IP of type `Host` reserves by name;
- **providers**, installed plugins that reserve IPs of type `Provider` and make load balancers.

The words (internal IP, external IP, service, port) are defined on
[Addresses, ports and names]({{ '/use/networking/model/' | relative_url }}).

---

## A host's addresses

A `Host` has its main `internalIP` and `externalIP`: a `HostPort` service on `worker: internal` or
`worker: external` opens on them, wherever the silo lands. Both are IPv4. A host with a global IPv6
address also has `externalIPv6`, and `worker: external` opens on it too
([IPv6 on a host](#ipv6-on-a-host)). A host can also have **extra addresses**,
routed to that machine, each with a name:

```yaml
kind: Host
metadata: { name: baremetal-1 }
spec:
  internalIP: 10.0.0.5
  externalIP: 203.0.113.10
  addresses:
    - name: mail
      kind: external          # or internal
      address: 203.0.113.21
    - name: mail6
      kind: external
      address: 2001:db8:20::21
```

Add one:

```bash
kvoadm host addresses baremetal-1 add mail 203.0.113.21 --external
```

```text
→ baremetal-1's extra addresses: mail 203.0.113.21
```

An IPv6 address is added the same way:

```bash
kvoadm host addresses baremetal-1 add mail6 2001:db8:20::21 --external
```

List them, with the IP that reserves each:

```bash
kvoadm host addresses baremetal-1 list
```

```text
NAME     KIND       ADDRESS        RESERVED BY
(main)   external   203.0.113.10   the worker's (hostPort: { worker: external })
(main)   internal   10.0.0.5       the worker's (hostPort: { worker: internal })
mail     external   203.0.113.21   ip mail/mail, used by silo/mail (25/TCP)
eth-0    external   203.0.113.22   -
```

Remove one:

```bash
kvoadm host addresses baremetal-1 remove eth-0
```

- One address each, IPv4 or IPv6. No ranges: add each address you want to hand out, even from an
  IPv6 `/64`. An IPv6 address is stored in its short form.
- An IP of type `Host` naming it is of its family (`family: IPv6` for an IPv6 address). Changing an
  address to the other family while an IP of the first names it is refused.
- Names are unique in the whole cluster, so an IP names one without saying which host.
- An address can't be on two hosts, and can't be the host's own internal or external IP.
- Removing an address an IP reserves is refused: delete that IP first.
- A silo using one runs on that host only. If the host is down, it waits for it: it can't move.

`kvoadm kvo describe host baremetal-1` lists the main and extra addresses, with who reserves and
uses each:

```text
Addresses:
  NAME    KIND      FAMILY  ADDRESS          RESERVED BY
  (main)  external  IPv4    203.0.113.10     used by mail/webmail (worker: external)
  (main)  external  IPv6    2001:db8:10::5   used by mail/webmail (worker: external)
  (main)  internal  IPv4    10.0.0.5         -
  mail    external  IPv4    203.0.113.21     ip mail/mail, used by silo/mail (25/TCP)
  mail6   external  IPv6    2001:db8:20::21  ip mail/mail6
  eth-0   external  IPv4    203.0.113.22     - (free: an IP with type: Host, hostAddress: eth-0 reserves it)
```

The host's `externalIPv6` is a `(main)` row of family `IPv6`. `kvoadm host addresses HOST list`
lists the IPv4 main addresses and every extra address.

The address must actually reach the machine: route it to the host at your provider or on your
network, and bring it up on an interface. Kevaso opens its ports and sends from it; it doesn't
configure your router.

### IPv6 on a host

A host's `externalIPv6` is its global IPv6 address. Kevaso finds it by itself: it is the address
the machine uses to reach the IPv6 internet. A host without one is IPv4 only, and `kvo describe
host` says so:

```text
External IP:        203.0.113.20
External IPv6:      none: IPv4 only (no global IPv6 address; set spec.externalIPv6 if it has one)
Silo subnets:       10.64.3.0/24 (no IPv6)
```

With one, the host also gets an IPv6 subnet for its silos, beside its IPv4 one:

```text
External IPv6:      2001:db8:10::5
Silo subnets:       10.64.2.0/24, fd12:3456:789a:2::/64
```

- `externalIPv6` takes one global IPv6 address only. A unique local (`fd00::/8`), link-local or
  loopback address is refused: such a host is IPv4 only.
- An IPv6 address is refused in `internalIP` and `externalIP`, and an IPv4 address in
  `externalIPv6`, each with the field to use. `kvoadm host add` takes IPv4 in `--internal-ip` and
  `--external-ip`.
- The silos' IPv6 subnet is private to the host, from a range Kevaso picks once for the cluster. It
  isn't routed anywhere: the host forwards to its silos and translates their outgoing traffic, as
  for IPv4. So one global IPv6 address is enough; no routed range is needed.
- An extra IPv6 address must reach the machine, as an IPv4 one does: routed to the host by your
  provider or your network.

What the machine needs is on [Prepare a machine]({{ '/admin/setup/prepare/#ipv6' | relative_url }}).

### Who may reserve them

There is no list of namespaces on an address. Who may use one is a permission, like every other:

| Action | Needs | Built-in roles |
|---|---|---|
| Make, change or delete an `IP` or `IPSet` (reserve a host's address, or a billed provider IP) | `create`, `update`, `delete` on `ips`, `ipsets` in the namespace | Namespace `admin` |
| Name an IP or IP set from a silo or a service | `use` on `ips`, `ipsets` (by name, if you like) | `editor`, `admin` |
| A `HostPort` or `LoadBalancer` service (anything reachable from outside) | `create` on `services/expose-public` | `admin` |
| Add or remove a host's extra addresses | Writing the `Host` | Cluster admin |

A role can be narrowed to one IP by name, as for secrets
([People and access]({{ '/admin/access/#roles' | relative_url }})). An operator who hosts other
people's silos and gives them only `Internal` services and [Ingress]({{ '/admin/ingress/' | relative_url }})
host names leaves `services/expose-public` and `ips` out of their roles: nothing they write then
reaches outside except through the ingress.

---

## Providers

A **provider** is an installed [plugin]({{ '/admin/extending/plugins/' | relative_url }}),
cluster-wide, like a storage class. It reserves IPs of type `Provider` and makes the balancers of
`LoadBalancer` services. A team names it by its install name:

```yaml
loadBalancer: { provider: linode }
```

```yaml
kind: IP
spec: { type: Provider, provider: linode, region: us-ord }
```

The install name is `linode` by default. A second account of the same provider is a second
install, under another name:

```bash
kvoadm plugin install root@203.0.113.10 --name linode-work --binary ./kevaso-provider-linode-linux-amd64 --cluster-secret linode-work
```

Its annotations then start with that name: `linode-work.kevaso.com/…`. With two installed, a team
always says which; Kevaso never picks one.

What a provider declares decides what Kevaso accepts and where silos go:

| It declares | So |
|---|---|
| It makes provider IPs | IPs of type `Provider` naming it are accepted (in the regions it serves), in the families it can reserve: IPv6 only when it declares `providerIPv6`. Linode reserves IPv4 only, and an IPv6 one is refused at apply with its reason |
| It makes load balancers | `LoadBalancer` services naming it are accepted. Whether a balancer answers on IPv6 is the provider's: Linode's do, shown as `loadBalancerIPv6` |
| A kept IP can front its balancer | `loadBalancer: { ip: NAME }` is accepted. Otherwise refused: the balancer has its own address |
| Its reach (`host`, `region`, `global`) | A silo using its IPs or balancers is placed and moved within it |
| Only its own machines | Only workers that are its machines can be behind its IPs and balancers |
| Its annotations, with their types | A misspelled annotation, or a wrong value, is refused at apply |

Each provider's page lists its annotations and limits: [Linode]({{ '/admin/providers/linode/' | relative_url }}).
Bare metal has no balancer provider yet: a `LoadBalancer` service has nowhere to go there, so use
`HostPort` services, on the worker's address or a host's extra address.

---

## What isn't done

- An external IP that is not on any of the host's interfaces (a cloud's 1:1 NAT address) matches
  nothing: a `HostPort` service on `worker: external` doesn't work there yet.
- Quotas on billed IPs and balancers are the provider's own, for now.
- A balancer provider for bare metal, and DNS that joins sites in different regions come later.
- The mesh is IPv4: IPv6 is at the edge only ([IPv6 at the edge]({{ '/use/networking/ipv6/' | relative_url }})).
