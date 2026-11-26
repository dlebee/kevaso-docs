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
`worker: external` opens on them, wherever the silo lands. A host can also have **extra addresses**,
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
```

Add one:

```bash
kvoadm host addresses baremetal-1 add mail 203.0.113.21 --external
```

```text
→ baremetal-1's extra addresses: mail 203.0.113.21
```

List them, with the IP that reserves each:

```bash
kvoadm host addresses baremetal-1 list
```

```text
NAME   KIND       ADDRESS        IP
mail   external   203.0.113.21   ip mail/mail (held by silo/mail (25/TCP))
eth-0  external   203.0.113.22   -
```

Remove one:

```bash
kvoadm host addresses baremetal-1 remove eth-0
```

- One address each, IPv4. No ranges: add each address you want to hand out.
- Names are unique in the whole cluster, so an IP names one without saying which host.
- An address can't be on two hosts, and can't be the host's own internal or external IP.
- Removing an address an IP reserves is refused: delete that IP first.
- A silo using one runs on that host only. If the host is down, it waits for it: it can't move.

`kvoadm kvo describe host baremetal-1` lists the main and extra addresses, with who reserves and
uses each:

```text
Addresses:
  NAME    KIND      ADDRESS       RESERVED BY
  (main)  external  203.0.113.10  used by mail/webmail (worker: external)
  (main)  internal  10.0.0.5      -
  mail    external  203.0.113.21  ip mail/mail, used by silo/mail (25/TCP)
  eth-0   external  203.0.113.22  - (free: an IP with type: Host, hostAddress: eth-0 reserves it)
```

The address must actually reach the machine: route it to the host at your provider or on your
network, and bring it up on an interface. Kevaso opens its ports and sends from it; it doesn't
configure your router.

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
| It makes provider IPs | IPs of type `Provider` naming it are accepted (in the regions it serves) |
| It makes load balancers | `LoadBalancer` services naming it are accepted |
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
- A balancer provider for bare metal, IPv6, and DNS that joins sites in different regions come later.
