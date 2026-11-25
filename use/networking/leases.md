---
title: Port leases
permalink: /use/networking/leases/
redirect_from: /networking/leases/
---

# Port leases

A **lease** keeps a port closed until someone opens it for a while. When the time is up, the
port closes by itself. Use it for a port you want open only now and then: a debug port, an
admin page, a management API while you check on it.

A lease belongs to one service port, and only to a port that opens outside its silo IP: one with
`hostPort` or `loadBalancer: true` (see [Addresses, ports and names]({{ '/use/networking/model/' | relative_url }})).
Silo-to-silo access is never leased: that is `allowFrom`, and it is always on.

---

## Declare the leased ports

`lease` goes on the service port. Each leased port has its own terms.

```yaml
spec:
  ports:
    - { name: web,   port: 8080 }
    - { name: debug, port: 6060 }
  service:
    ports:
      - { port: web, hostPort: [external] }          # always open
      - port: debug
        hostPort: [internal]
        sources: ["203.0.113.0/24"]
        lease:
          default: 15m
          max: 2h
          approval: { required: true }
```

| Field | Meaning |
|---|---|
| `default` | The length when the ask names none. Default: 30 minutes, or the max if that is shorter. |
| `max` | The longest this port may ever get. Empty: the cluster's default limit (below). |
| `approval.required` | Each lease of this port needs a person's yes first. |

- A port with a `lease`, even `lease: {}`, is **closed until leased**. A port without one is always
  open.
- Who may connect while a lease is open is the port's own `sources` (and, on the host, the host
  address it opens on). A lease never makes it wider.
- `lease` on a port that doesn't open outside its silo IP is refused: silo-to-silo access is never
  leased.
- Durations are written like `90s`, `30m` or `2h`, at least one second. `default` can't be more than
  `max`, and `max` can't be more than the cluster's limit.

`kvo apply` tells you if a value breaks a rule, and why. The same field is allowed in a SiloSet's
template.

---

## Open and close

### From your computer

```bash
kvo open silo node-1 --port debug --for 1h
```

```text
silo/node-1 port debug: open until 4:12PM
```

```bash
kvo close silo node-1 --port debug
```

| Flag | Meaning |
|---|---|
| `--port NAME` | The leased port, by its declared name (required; repeat it for several ports) |
| `--for DURATION` | How long. Empty: the port's `lease.default`. More than the port's max: you get the max, and `kvo` says so |
| `--no-wait` | With `kvo open`: if a lease needs a yes, print what to approve and return |

With several `--port` flags, each port is its own ask on its own terms: its own lease, its own max
and, if it needs one, its own approval. `kvo` sends all the asks, then waits for the ones that need a
yes. A refused port is reported on standard error and doesn't stop the others. The command then
exits non-zero, with a line such as `silo/node-1: 1 of 2 ports not opened: debug`.

```bash
kvo open silo node-1 --port debug --port preview --for 30m
```

Opening needs the verb `open` on `silos/exposure` ([below](#who-may-do-what)).

### From inside the silo

The silo can ask for its own port, over its [local endpoint]({{ '/use/secrets/' | relative_url }}). A silo with a leased
port has one:

```bash
curl -fsS -H 'Kevaso-Local: 1' -X POST 'http://169.254.42.1/v1/ports/debug/open?for=1h'
```

```bash
curl -fsS -H 'Kevaso-Local: 1' -X POST http://169.254.42.1/v1/ports/debug/close
```

| Query parameter | Meaning |
|---|---|
| `for=DURATION` | How long, as above |
| `wait=DURATION` | If it needs a yes: how long the request may wait for one |

A successful open answers `{"port":"debug","until":"…","clamped":false}`. If the lease needs a yes,
the answer is HTTP `428` with the pending approval, and the silo asks again with `approval=NAME`
until it is answered (see [approvals]({{ '/use/secrets/' | relative_url }})). A
refusal is `403`. To open several ports, call once for each. A silo can only open and close **its
own leased ports**: the silo that asks is known from where the request arrived.

---

## A person's yes

With `approval.required: true`, an ask doesn't open the port. It becomes an
[approval]({{ '/use/secrets/' | relative_url }}) with the use `expose`, and the port stays closed:

```bash
kvo open silo node-1 --port debug --for 1h
```

```text
silo/node-1 port debug: waiting for a yes: kvo approve node-1-debug-3f9a1c -n infra (expires 4:12PM)
```

Someone else answers. The `REQUEST` column and `--request` are the same as for a secret: [approvals]({{ '/use/secrets/#ask-a-person-first-approvals' | relative_url }}).

```bash
kvo get approvals -n infra
```

```bash
kvo approve node-1-debug-3f9a1c -n infra --request 3f9a1c2e4b7d6a0c8e1f2a3b4c5d6e7f
```

```bash
kvo deny node-1-debug-3f9a1c -n infra --reason "not today"
```

- **Only people answer**, with the verb `approve` on `silos/exposure`. A silo's own yes is never
  taken.
- **Someone who holds `approve` there, and asks, needs nobody's yes.** Their own ask opens at once.
- **The yes names the port.** The approval says which port, from which sources, and the port's max.
- **A waiting ask always expires.** Nobody answered in time: the port stays closed.
- **Asking again replaces** the waiting ask.
- **Extending is a new ask.** A port that is open stays open while an extension waits for its yes.
- The lease counts from the moment of the yes.

---

## What a lease limits

Two limits, set on the control plane, and the silo's own `lease.max` between them:

| Flag of `kevasod control-plane` | Default | Means |
|---|---|---|
| `--lease-max` | `24h` | The longest a port may ever be open on a lease. A silo whose `lease.max` is more than this is refused when applied. |
| `--lease-default-max` | `1h` | The longest lease of a silo whose `lease` sets no `max` |

An ask for more than the max is not an error: it gets the max. The answer says it was cut.

To change them, edit the service on the control-plane host (`sudo systemctl edit --full kevasod`),
add the flags to the `ExecStart` line of `kevasod control-plane …`, and restart.

---

## When a lease ends

At the end of its time, or on `kvo close`:

- The host takes away the port's rules (the forward, or the node port), so **new connections are
  refused**.
- Connections already open on that port are **reset**, so nothing carries on through a lease that
  has ended.
- Hosts don't wait for the control plane to say so: each worker wakes at the next lease's end by
  itself, even if the control plane is down or out of reach.
- On a [load balancer from a plugin]({{ '/admin/extending/plugins/' | relative_url }}), the port is taken out of what the plugin is asked to serve, and that is reconsidered at the end of the lease.

A silo that is deleted and made again with the same name starts with its ports closed: a lease
belongs to one silo, not to a name.

---

## See them

```bash
kvo get leases
```

```text
SILO     PORT    PHASE    WHEN                   BY
node-1   debug   Open     until 4:12PM           bob
node-1   admin   Closed   time up                alice (yes: carol)
```

| Phase | Means |
|---|---|
| `Open` | Open until the time shown. An extension may be waiting |
| `Pending` | An ask waits for a yes; the port is closed |
| `Closed` | Time was up, or someone closed it |
| `Denied` | The last ask was denied |
| `Expired` | The last ask's approval wasn't answered in time |

`kvo describe silo node-1` has an **Outside ports** table: each port that opens outside the silo IP,
with its lease terms (or `-`) and its state, `always open`, `closed`, `pending` or `open until` a time:

```text
Outside ports:
  PORT     LEASE                                            STATE
  web      -                                                always open
  debug    default 15m, max 2h, a person's yes first, from 203.0.113.0/24   open until 4:12PM
```

There is one
`Lease` per leased port, named `SILO-PORT-xxxxxx`. Only the control plane writes them; you can't
create or edit one, and an admin can delete one.

Every step is in the [audit log]({{ '/admin/operations/#the-audit-log' | relative_url }}) with the
resource `silos/exposure`: `lease-asked`, `lease-replaced`, `lease-opened`, `lease-denied`,
`lease-expired`, `lease-closed`, `lease-ended`, `lease-refused`.

---

## Who may do what

| Resource | Verb | Lets you | `viewer` | `editor` | `admin` |
|---|---|---|---|---|---|
| `silos/exposure` | `open` | `kvo open silo`, `kvo close silo` | | yes | yes |
| `silos/exposure` | `approve` | Answer an `expose` approval | | | yes |
| `leases` | `get`, `list`, `watch` | `kvo get leases` | yes | yes | yes |
| `leases` | `delete` | Delete a lease | | | yes |

`open` and `approve` can be given for one silo, by its name or by label, like any other rule:
see [People and access]({{ '/admin/access/' | relative_url }}#roles).

Opening a port to the outside, or widening one that is already open, is still its own permission,
`services/expose-public`: a lease only decides *when*.
[What counts as widening]({{ '/admin/access/#roles' | relative_url }}).

---

## Load balancers from a plugin, and `sources`

A port's `sources` say which network ranges may connect, leased or not. Some load balancers can't
filter by client address. The rule:

| Class | What happens to a port that has `sources` |
|---|---|
| Announces its address (a plugin class with `announce`: the address is routed to the silo's host, which sees the client's own address) | The host's firewall enforces it |
| A plugin's node-port class with `filtersSources: true` | The plugin is given each port's sources and must enforce them |
| A plugin's node-port class without it | **The port is left out.** The silo's `LoadBalancer` condition says `SourcesNotEnforced` and names the ports |

The last row closes a hole rather than leaving one: a port someone narrowed is never served to
everyone. If you hit it, drop the `sources`, use a host port or an announced class, or use a
plugin that filters. Plugin authors: see [Plugins]({{ '/admin/extending/plugins/' | relative_url }}).

---

## Not done

- A lease on an **Ingress** host. Only service ports with `hostPort` or `loadBalancer: true` lease.
