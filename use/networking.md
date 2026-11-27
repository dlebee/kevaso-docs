---
title: Networking
permalink: /use/networking/
redirect_from: /networking/
---

# Networking

Kevaso owns the firewall on its hosts, and starts from **deny**. A silo declares its ports, and a
**service** opens some of them, in one of three ways. This page goes through each way in, where a
silo using it can run, and which address its traffic leaves from.

The host firewall is the administrator's to watch: when a host can't load a new ruleset, the last
one stays in force ([Operations]({{ '/admin/operations/#troubleshooting' | relative_url }})).

---

## The words used on this page

Internal IP, external IP, mesh IP, silo IP and cluster IP each mean one thing in Kevaso, and so do
"port", "service" and a `.kevaso` name. They're defined, with pictures, in
[Addresses, ports and names]({{ '/use/networking/model/' | relative_url }}). Addresses you keep, the
`IP` and `IPSet` kinds, are on [IPs and IP sets]({{ '/use/networking/ips/' | relative_url }}). What
answers on IPv6, and what stays IPv4, is on [IPv6 at the edge]({{ '/use/networking/ipv6/' | relative_url }}).

---

## The defaults

| From → to | Default | Opened by |
|---|---|---|
| A silo → another silo | Denied | The receiving port's `allowFrom` |
| Outside → a silo | Denied | A service of type `HostPort` or `LoadBalancer` |
| A silo → outside | Denied | A `Grant` naming the destination ([below](#what-a-silo-may-reach-outside)) |
| A silo → its host | DNS, and ping from a microVM, on this host's gateway | Never more, except the [local endpoint]({{ '/use/secrets/' | relative_url }}) for a silo that has something to ask it, and the egress gate for a silo whose grants name hosts. A container silo cannot ping: it has no raw socket ([Runtimes]({{ '/use/silos/runtimes/#docker-and-podman' | relative_url }})) |
| A silo → another host's own listeners, over the mesh | Denied | Never. A packet from a silo that arrives on the mesh is dropped |
| A silo → any host's `10.64.N.1` | Denied | Never, except this host's own DNS (port 53) and ping from a microVM |
| A silo → names in the cluster | Allowed | `.kevaso` names, and short service names (`api`, `api.payments`) |
| A silo → a name outside the cluster | Only when the silo lists a Grant that exists | With no Grant, the name is refused |

Each silo also can't send packets with an address other than its own (anti-spoofing), so a rule
"from silo X" really means silo X. A packet with a source in the silo range is dropped unless it
arrives on the bridge or the mesh.

---

## Declare a port, then open it

A silo declares its ports in `spec.ports`. Its `services` list opens them. Each service is **one way
in**, and has its own name, `NAME.NAMESPACE.kevaso`:

```yaml
spec:
  ports:
    - { name: http,    port: 8080 }
    - { name: https,   port: 8443 }
    - { name: metrics, port: 9090 }
  services:
    - name: web
      type: HostPort                       # no hostPort: means { worker: external }
      ports: [{ port: http, as: 80 }]
    - name: api
      type: HostPort
      hostPort: { ip: api-edge }           # an IP you keep, in this namespace
      ports: [{ port: https, as: 443 }]
    - name: shop
      type: LoadBalancer
      loadBalancer: { provider: linode }   # a balancer from an installed provider
      ports: [{ port: https, as: 443 }]
    - name: metrics                        # type: Internal is the default
      ports: [{ port: metrics, allowFrom: [monitoring/prometheus] }]
```

- `spec.ports` says what exists. **Nothing is reachable unless a service lists it.** A silo with no
  service has no name and nothing reachable: it can still call out.
- A service with no `name` takes the silo's name. Two unnamed services in one silo clash: name them.
- A port can be in several services (`https` above is in two). Everything about one port inside one
  service is in one entry.
- One service has one way in. A port you want on the worker's internal address and behind a balancer
  is in two services, one of each type. Giving one service two (`hostPort` on a `LoadBalancer`, or
  `worker` and `ip` together) is refused, with the fix:

```text
service api: hostPort takes exactly one of worker, ip and own (two ways in inside one service): make a service for each
```

### The three ways in

| `type` | Traffic arrives | On the address | Where the silo can run |
|---|---|---|---|
| `Internal` (the default) | Over the mesh, from the silos in `allowFrom` | The silo IP | Anywhere |
| `HostPort`, `hostPort: { worker: external }` (the default) or `{ worker: internal }` | At the worker running the silo | That worker's main external or internal IP | Any worker that has that address. It changes when the silo moves |
| `HostPort`, `hostPort: { ip: NAME }` | At the worker holding the IP | An [`IP`]({{ '/use/networking/ips/' | relative_url }}) you keep | Type `Host`: on that host only. Type `Provider`: within the provider's reach (Linode: its region, on Linode workers) |
| `HostPort`, `hostPort: { own: true }` | At the worker holding the instance's address | A SiloSet instance's member of its [`IPSet`]({{ '/use/networking/ips/#ip-sets' | relative_url }}) | As for that member's IP |
| `LoadBalancer`, `loadBalancer: { provider: NAME }` | At a balancer the provider makes, in front of the worker | The balancer's address | Within the provider's reach |

`allowFrom` works with every type: it is always "which silos may connect on the silo IP".

### Letting another silo in: `allowFrom`

`allowFrom` on a service port names the silos that may connect to it on its silo IP: `NAME` for the
same namespace, `NAMESPACE/NAME` for another. It works across hosts too, over the mesh. A silo named
here that isn't running opens nothing. `allowFrom` takes silos only: a network range there is
refused, and ranges go in `sources` on a `HostPort` or `LoadBalancer` service.

Removing an `allowFrom` entry also cuts the open connections that entry allowed. TCP is reset.
UDP is dropped. The cut stays in later rulesets, so that connection does not resume on the next
reload.

### At the worker: `HostPort`

A `HostPort` service opens its ports on one address of the worker running the silo, like a
Kubernetes `hostPort` (not a `NodePort`: only the silo's own worker answers). `hostPort` says which
address, exactly one of:

| `hostPort` | The address | Written in |
|---|---|---|
| Left out, or `{ worker: external }` | The worker's main external IP, and its IPv6 address (`externalIPv6`) when it has one | A silo or a SiloSet's template |
| `{ worker: internal }` | The worker's main internal IP (IPv4 only) | A silo or a SiloSet's template |
| `{ ip: NAME }` | The address of the `IP` named `NAME` in this namespace, in that IP's family | A silo only |
| `{ own: true }` | This instance's member of the set's `IPSet` | A SiloSet's template only, with `ips: SETNAME` on the set |

On each port, `as` is the number there when it differs from the silo's, and `sources` limits who may
connect to network ranges. A `lease` keeps the port closed until someone opens it for a while
([Port leases]({{ '/use/networking/leases/' | relative_url }})); leases are on `HostPort` services only.

It is plain layer-4 forwarding: no proxy, and **the silo sees the client's real address**. The host
never answers a host port on Kevaso's own networks (the mesh, the silos, the Services' virtual IPs).

One silo holds a host port on a host: one address, protocol and number. The store records the
holder. A conflicting write of the silo keeps the host ports that silo already holds. A second silo
does not get that port opened. On the firewall the port opens only for the silo that holds it; two
askers, and it opens for nobody. A silo that was placed without claiming, while another silo holds
the port, has `HostPorts` false, reason `HeldByAnother`:

```text
host ports external-tcp-80 on hv1 are held by another silo (it was put there without claiming them); move one of the two
```

A silo may not take a port the platform listens on, on TCP or UDP. 80 and 443 are not in this list.

| Port | Why |
|---|---|
| 22 | the SSH port |
| 53 | DNS |
| 2379, 2380 | etcd |
| 6443 | the API |
| 51820, 51821 | the mesh |
| 15001 | the egress gate |

A host that listens elsewhere adds its own: its SSH port when that is not 22, its mesh listen port,
and the API and etcd ports it publishes. The refusal names the reason:

```text
service web: ports[web]: 6443 is the API, which a silo may not take
```

```text
Silo web: port 2222 is the SSH port, which a silo may not take
```

A standalone `Service` (a group of silos by label) can't be a `HostPort`: one address at one worker
can't front several silos. Give each silo its own `HostPort` service (in a SiloSet, `own: true` for
an address each), or use a `LoadBalancer` Service for the group.

### Behind a balancer: `LoadBalancer`

A `LoadBalancer` service asks an installed **provider** for a balancer in front of the silo:

```yaml
services:
  - name: shop
    type: LoadBalancer
    loadBalancer: { provider: linode }
    ports: [{ port: https, as: 443 }]
```

- **One balancer per service**, made with the service and removed with it. Sharing one balancer
  between services, its size and its tuning are the provider's **annotations** on the service,
  checked when you apply: a misspelled one is refused. Each provider's page lists them
  ([Linode]({{ '/admin/providers/linode/#a-nodebalancer-in-front' | relative_url }})).
- The balancer reaches the silo through a port on its worker that is open to the balancer's
  addresses only. Its backends are the silos that are ready ([`routeWhen`]({{ '/use/silos/probes/' | relative_url }})).
  Whether the silo sees the client's own address is the provider's to say.
- `loadBalancer: { provider: NAME, ip: IPNAME }` puts an [IP you keep]({{ '/use/networking/ips/' | relative_url }})
  in front of the balancer, only where the provider allows it. A Linode NodeBalancer has its own
  address, so on Linode it is refused.
- A provider is installed by an administrator, under a name: `linode` by default, another name for a
  second account ([Host addresses and providers]({{ '/admin/addresses/#providers' | relative_url }})).
  Naming one that isn't installed is refused, and the refusal lists the ones that are.
- A silo with several `LoadBalancer` services: `kvo describe silo` shows the first one's address
  for now. Each balancer still works on its own address.
- On a balancer that can't limit clients by address, a port with `sources` is left out, not opened to
  everyone: see [leases and sources]({{ '/use/networking/leases/' | relative_url }}#balancers-and-sources).

A standalone `Service` of type `LoadBalancer` is one balancer for a group of silos: see
[Sets]({{ '/use/silos/sets/' | relative_url }}#an-outside-address). In a SiloSet's template, a
`LoadBalancer` service makes one balancer per instance, each billed, and `kvo apply` warns you.

### Plain HTTP and HTTPS sites

For HTTP and HTTPS by host name and path, with certificates, use an [ingress]({{ '/use/ingress/' | relative_url }}).
The silos behind it stay private.

---

## Who may write what

| Written in | May use | Refused |
|---|---|---|
| A `Silo` | `Internal`; `HostPort` with `worker` or `ip`; `LoadBalancer` | `own`; an `IPSet` |
| A `SiloSet`'s template | `Internal`; `HostPort` with `worker` or `own` (and `ips: SETNAME` on the set); `LoadBalancer` (one per instance, warned) | `ip`: one IP can't serve many instances, even with one instance |
| A standalone `Service` (a group, by label) | `Internal`; `LoadBalancer` | `HostPort` |
| A silo with several services | One way in per service; a port in several services | Two ways in inside one service |

> **Opening a port to the outside is its own permission:** `create` on `services/expose-public` in
> the namespace. A `HostPort` or `LoadBalancer` service needs it, in a silo, a SiloSet's template or
> a standalone Service. An edit that widens that reach needs it too
> ([People and access]({{ '/admin/access/#roles' | relative_url }})). Namespace `admin`s have it,
> `editor`s don't. Naming an IP (`ip: NAME`) or an IP set (`ips: SETNAME`) needs `use` on it, like a
> secret: editors and admins have it.

---

## Where a silo can run

**Placement follows the address.** Each way in narrows where the silo can run, and a silo with
several ways in runs where all of them can:

| Way in | Where |
|---|---|
| `Internal`, `worker: external`, `worker: internal` | Any worker with that address. The address is the worker's, so it changes when the silo moves |
| An `IP` of type `Host` (`ip`, or an `own` member that is a host's address) | On that host only |
| An `IP` of type `Provider` | Within the provider's reach (Linode: workers that are Linodes, in the IP's region). After a move, the new worker announces the address |
| A `LoadBalancer` | Within the provider's reach |

`kvo describe silo` says where it can go, and why:

```bash
kvo describe silo api-blue -n payments
```

```text
Can move:   yes: within linode/us-ord (IP api-edge)
```

On a worker's own address it is `yes: any worker`. A silo pinned by a host's address says so:

```text
Can move:   no: pinned to baremetal-1 (IP mail is that host's address)
```

A move to a worker outside that area is refused, and the refusal names the address that holds the
silo where it is ([Moves]({{ '/use/silos/moves/' | relative_url }})).

### Refused, or waiting

Kevaso tells "can never work" from "could work later":

| Kind | Examples | What happens |
|---|---|---|
| **Can never work** | An `ip` in a SiloSet's template; `own` outside a SiloSet; two ways in inside one service; a `Dynamic` IP set with a host's address; two `Host` IPs on different hosts in one silo; a misspelled provider annotation; an IP whose `type` doesn't match its fields | Refused when you apply, with the fix |
| **Could work later** | No free member in the IP set; the provider's quota; an IP held by another silo; no worker with an external IP; a Mac with no public IP | The silo waits, unscheduled, and says why. It goes ahead by itself when that changes |
| **Ambiguous** | Two regions, two provider accounts | Refused, naming the word to add. Kevaso never guesses |

For example, a SiloSet's template naming one IP:

```text
template.services[0].hostPort.ip names IP api-edge: one IP can't serve many instances (even with one instance): use own: true and put the set on an IPSet (ips: SETNAME)
```

---

## Which address its traffic leaves from

By default a silo's outgoing traffic leaves from **its worker's own IP**. `network.egress.source`
sends it from an address you keep instead, usually the one it receives on:

```yaml
spec:
  services:
    - type: HostPort
      hostPort: { ip: mail }
      ports: [{ port: smtp }]
  network:
    egress: { source: { ip: mail } }     # out from the IP it receives on
```

| `egress.source` | Outgoing traffic leaves from | Written in |
|---|---|---|
| Left out | The worker's own IP (its IPv6 address for IPv6 destinations). A worker behind NAT has none to name | Anywhere |
| `{ ip: NAME }` | The `IP` named `NAME`, for destinations of its family. The other family still leaves from the worker | A silo |
| `{ own: true }` | The instance's member of the set's `IPSet` | A SiloSet's template |

Can a silo use the same IP out as in?

| It is reached on | Same address out? |
|---|---|
| Nothing (no service), or `Internal` only | Nothing to match: it sends from the worker's IP |
| `HostPort`, `worker` | **Yes**, by default: that address is the worker's own |
| `HostPort`, `ip` or `own` | **Yes**, with `egress.source` naming the same IP (or `own: true`) |
| A `LoadBalancer` | **No.** The balancer's address is on no worker, so nothing can send from it |
| An [Ingress]({{ '/use/ingress/' | relative_url }}) controller | **No.** The address belongs to the controller, not to your silo |

To see both addresses for a running silo:

```bash
kvo describe silo mail -n mail
```

```text
Egress IP:   203.0.113.21 (ip: mail)
In and out:  same address in and out: 203.0.113.21
```

`Egress IP` is where its outgoing traffic leaves from; when you set `source`, it is in brackets after
the address. `Egress IPv6` is the same for IPv6 destinations: the worker's IPv6 address, or a kept
IPv6 IP that `source` names, and `none` on a worker without IPv6.

`In and out` says in one line whether the address out is the address in, and why not
when it isn't. A `-` means there is none, for example on a worker behind NAT.

Where it may send is still a [Grant](#what-a-silo-may-reach-outside)'s to say: an address to send
from opens nothing.

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
    - { to: "2001:db8:44::/48", ports: ["443"] }               # an IPv6 range
---
apiVersion: kevaso/v1
kind: Silo
metadata: { name: builder, namespace: dev }
spec:
  runtime: cloud-hypervisor
  vm: { image: ubuntu-24.04 }
  grants: [build-tools]
```

An OpenShell sandbox (`openshell`, `openshell-container`) has no network card. A grant opens a host
and a port. An [OpenShellPolicy]({{ '/use/silos/runtimes/#openshell-policies' | relative_url }}) adds a
path, read-only or read-write access, REST, and which programs may use a rule.

| `to:` | How it's enforced |
|---|---|
| An address or CIDR | The host firewall, by address, port and protocol |
| A host name | The host's **egress gate**: the silo's TLS is checked against the name it asks for (SNI) and passed through, untouched, only for that name, to the address the gate looks up itself. No name or another name: refused. A name opens TCP only, port 443 unless you list ports. A name that resolves to a private, loopback or link-local address, or to the mesh, a silo, a service or a host in the cluster, is refused, and so is that address written as IPv6. To allow one, a grant has to name that address, or a range no wider than that block that contains it, for that port. A rule for every address does not. (The gate doesn't look inside a passed-through connection; for hosts it injects a secret for, it does, and refuses a request for another host: see [Secrets]({{ '/use/secrets/' | relative_url }}).) |

A rule opens TCP unless it lists `protocols` (`tcp`, `udp`; a host name is TCP only). `ports` takes
numbers and ranges such as `"30000-30100"`; with none, an address range opens every port and a host name
opens 443. Addresses and ranges can be IPv4 or IPv6. A host name connects over IPv4: write the
address or range to reach a destination over IPv6 ([IPv6 at the edge]({{ '/use/networking/ipv6/#sending-to-ipv6' | relative_url }})).
An IPv4 address written inside IPv6 (`::ffff:203.0.113.7`) is refused: write the IPv4 address.

**What a grant never opens.** Whatever the rule, even `0.0.0.0/0`, a silo never reaches:
- the host's loopback (`127.0.0.0/8`, `::1`), `0.0.0.0/8`, `::`, link-local (`169.254.0.0/16`, the
  cloud metadata address `169.254.169.254` included, and `fe80::/10`), multicast (`ff00::/8` too)
  or reserved addresses;
- the cluster's own ranges (the mesh, silo and service ranges, and the silos' IPv6 range) except as
  its silos allow;
- its own host's internal and external addresses;
- every host's address on the silo network, `10.64.N.1`, except this host's own DNS and ping;
- another host's own listeners, reached over the mesh. A silo on one host reaches nothing of another
  host itself. It still reaches that host's silos, as those silos allow.

A grant naming an address in the first two is refused when you write it. So `0.0.0.0/0` means "the internet and
your private networks", nothing more, and `::/0` the same for IPv6. A silo still reaches the ports its host publishes for other
silos, and its own host's address at `169.254.42.1` for [secrets]({{ '/use/secrets/' | relative_url }}).

On a port where the same silo also has a host-name rule, its connections go through the egress
gate, which also refuses every host of the cluster by address. To reach one there, such as an
Ingress or a load balancer on another host, the grant has to name that address itself (a `/32`).

Removing a Grant, or narrowing one, does not cut connections that are already open. It applies to
new connections. (An `allowFrom` entry is different: removing one cuts the connections it allowed,
[above](#letting-another-silo-in-allowfrom).)

A silo that lists no Grant, or only names that are not there, cannot resolve a name outside the
cluster. The answer is refused, and the host's resolvers are not asked. Names in the cluster still
work: `api.payments.kevaso`, and the short names `api` (same namespace) and `api.payments`. A silo
that lists a Grant that exists is forwarded to the host's resolvers for other names. Resolving a
name is not permission to connect: the grant still decides that.
