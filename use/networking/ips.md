---
title: IPs and IP sets
permalink: /use/networking/ips/
---

# IPs and IP sets

Some addresses must outlive the silo using them: a peer allow-lists it, a DNS record points at it,
a mail server's reputation is tied to it. An **IP** is one address kept as an object of its own, in
a namespace. An **IP set** is one address for each instance of a SiloSet.

A silo uses them in a `HostPort` service (`hostPort: { ip: NAME }`) and to send from
(`egress.source: { ip: NAME }`): see [Networking]({{ '/use/networking/' | relative_url }}). This page is
about the addresses themselves.

---

## An IP

An IP has an explicit `type`, which says where its address comes from:

```yaml
apiVersion: kevaso/v1
kind: IP
metadata: { name: mail, namespace: mail }
spec:
  type: Host                  # one of a host's extra addresses: pinned to that host
  hostAddress: mail
---
apiVersion: kevaso/v1
kind: IP
metadata: { name: api-edge, namespace: payments }
spec:
  type: Provider              # reserved from a provider: follows the silo within its reach
  provider: linode
  region: us-ord
```

| `type` | Needs | The address | A silo using it runs |
|---|---|---|---|
| `Host` | `hostAddress`: the name of one of a host's extra addresses | That address, on that host | On that host only |
| `Provider` | `provider` (an installed provider) and `region` | One the provider reserves for you, billed by it | Within the provider's reach: on Linode, workers that are Linodes in that region |

- The other type's fields are refused: an IP of type `Host` with a `region`, or of type `Provider`
  with a `hostAddress`, says which to drop.
- One IP is exactly one address. There are no ranges.
- A host's extra addresses are named by an administrator, with
  `kvoadm host addresses` ([Host addresses and providers]({{ '/admin/addresses/' | relative_url }})).
  Ask yours which names you may reserve.
- `provider` and `region` are never guessed: with two accounts or two regions, the refusal names the
  word to add.

### Make one

Apply the YAML above, or:

```bash
kvo create ip mail -n mail --type Host --host-address mail
```

```bash
kvo create ip api-edge -n payments --type Provider --provider linode --region us-ord
```

```text
ip/api-edge created: its address shows in kvo get ips once reserved
```

Making, changing and deleting an IP is a namespace `admin`'s. Naming one from a silo needs `use` on
it, like a secret: `editor`s and `admin`s have it
([People and access]({{ '/admin/access/#roles' | relative_url }})).

### See them

```bash
kvo get ips -n payments
```

```text
NAME       TYPE      ADDRESS       WHERE              SET   USED BY                   MOVES                    AGE
api-edge   Provider  172.232.0.9   linode us-ord      -     silo/api-blue (443/TCP)   within linode us-ord     3d
mail       Host      203.0.113.21  host baremetal-1   -     -                         no: pinned to its host   3d
```

| Column | Meaning |
|---|---|
| `TYPE` | `Host` or `Provider` |
| `ADDRESS` | The address, once reserved |
| `WHERE` | The host, or the provider and region |
| `SET` | The IP set it is a member of, and its ordinal (`edge#1`), if any |
| `USED BY` | The silo or service holding it, with its ports |
| `MOVES` | Where a silo holding it can go |

```bash
kvo describe ip api-edge -n payments
```

```text
Type:       Provider (linode, region us-ord)
Address:    172.232.0.9
Where:      linode us-ord
Moves:      with its holder, within linode us-ord (the new worker announces it)
Used by:    silo/api-blue (443/TCP)
Waiting:    silo/api-green: held by silo/api-blue
Deleting:   refused while silo/api-blue uses it: delete that one first, or have it name no IP
```

It shows the address, its type and where it comes from, the set and ordinal it belongs to, who
uses it, any silo waiting for it and why, whether it can be deleted now, and, for the silo using
it, whether that silo sends from it too.

### Kept until you delete it

- **Deleting the silo keeps the IP.** The next silo naming it gets the same address: run the new
  version as a second silo, then move the IP to it by changing which silo names it (blue/green).
- **One holder at a time.** A second silo naming an IP that another holds waits, and says which silo
  holds it. It goes ahead when the first lets go.
- **Deleting an IP something holds is refused**, naming the holder. Delete or change the holder first.
- A `Provider` IP follows its silo: after a move within the provider's reach, the new worker
  announces the address, and it answers there once the provider has moved it (on Linode, about a
  minute). A move out of its reach is refused, naming the IP.
- Deleting the namespace releases its `Provider` IPs, after the silos and balancers using them
  ([Namespaces]({{ '/use/namespaces/' | relative_url }})).

---

## IP sets

A SiloSet's instances can't share one IP. An **IP set** gives each instance one address of its own,
matched by ordinal: instance 0 gets member 0, instance 1 gets member 1, and so on (like a
StatefulSet's volume claims).

```yaml
apiVersion: kevaso/v1
kind: IPSet
metadata: { name: eth-ips, namespace: web3 }
spec:
  type: Static
  members:
    - { hostAddress: eth-0 }
    - { hostAddress: eth-1 }
---
apiVersion: kevaso/v1
kind: IPSet
metadata: { name: edge, namespace: payments }
spec:
  type: Dynamic
  provider: linode
  region: us-ord
  max: 5
```

| `type` | Members | Grows |
|---|---|---|
| `Static` | The list you write, in order, each `{ ip: NAME }` (an IP in this namespace) or `{ hostAddress: NAME }` (a host's extra address) | Never by itself. Add members at the end: they take the next ordinal, and nothing is renumbered |
| `Dynamic` | Provider IPs Kevaso reserves as instances need them, named `SET-0`, `SET-1`, … and kept per ordinal | Up to `max` |

- A `Dynamic` set takes `provider`, `region` and `max` (at least 1) and no members. A `Dynamic` set
  with a host's address is refused: list host addresses in a `Static` set.
- A `Static` set takes members and nothing else. Each member is exactly one of `ip` and
  `hostAddress`, and the same address twice is refused.
- Every member is an ordinary IP: `kvo get ips` lists them, with the set in `SET`. A
  `hostAddress` member's IP is named after the set and its ordinal (`eth-ips-0`), like a
  `Dynamic` set's.

```bash
kvo get ipsets -n web3
```

```text
NAME      TYPE     MEMBERS   USED   MAX   AGE
eth-ips   Static   2         2      2     3d
```

```bash
kvo describe ipset eth-ips -n web3
```

```text
Type:       Static (the members listed: it never grows on its own; add members at the end)
Used by:    siloset/eth-pool
Members:
  ORDINAL  MEMBER              IP         ADDRESS       USED BY
  0        hostAddress: eth-0  eth-ips-0  203.0.113.22  eth-pool-0
  1        hostAddress: eth-1  eth-ips-1  203.0.113.23  eth-pool-1
```

A SiloSet names its set with `ips`, and its template says where each instance uses its member:

```yaml
kind: SiloSet
spec:
  ips: eth-ips
  template:
    services:
      - type: HostPort
        hostPort: { own: true }
        ports: [{ port: p2p }]
    network:
      egress: { source: { own: true } }
```

The whole example is on [Sets]({{ '/use/silos/sets/#an-address-for-each-instance' | relative_url }}).

### When instances come and go

| Event | What happens |
|---|---|
| More instances than a `Static` set has members | The extra instance waits: `no free member in IPSet eth-ips: instance eth-pool-2 takes member 2, and it has 2 (add one at the end)` |
| A `Dynamic` set at `max`, or the provider out of addresses or quota | The instance waits, with `…, and its max is 5` or the provider's words |
| Scaling in | The members stay reserved: when instance 2 comes back, it gets member 2 again |
| Deleting or replacing the SiloSet | The IP set and its IPs stay. A new set naming it gets the same addresses in the same order |
| Deleting a `Static` set | The IPs you listed (`ip:` members) stay. The IPs it made for `hostAddress` members are deleted, and those addresses are free again |
| Deleting a `Dynamic` set | The IPs it reserved are released |
| Deleting a set while an instance still uses a member | The delete waits, `Deleting`, until that silo lets go of it |

Making an IP set is a namespace `admin`'s; naming one with `ips` needs `use` on it, as for an IP.
