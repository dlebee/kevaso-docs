---
title: Port leases
permalink: /networking/leases/
---

# Port leases

A **lease** keeps a port closed until someone opens it for a while. When the time is up, the
port closes by itself. Use it for a port you want open only now and then: a debug port, an
admin page, a node's RPC while you check on it.

A lease sits on top of [`hostPort` or `loadBalancer`]({{ '/networking/' | relative_url }}#open-to-the-internet).
It is never for silo-to-silo access: that is `allowFrom`, and it is always on.

---

## Declare the leased ports

```yaml
spec:
  service:
    ports:
      - { name: p2p,   port: 30303 }
      - { name: debug, port: 6060 }
    expose: hostPort
    lease:
      ports: [debug]             # only this one is leased; p2p stays open
      default: 15m
      max: 2h
      approval: { required: true }
      allowFrom: ["203.0.113.0/24"]
```

| Field | Meaning |
|---|---|
| `ports` | The leased ports, by name. Empty: every exposed port. A port that isn't exposed is refused. |
| `default` | The length when the ask names none. Default: 30 minutes, or the max if that is shorter. |
| `max` | The longest this silo may ever get. Empty: the cluster's default limit (below). |
| `approval.required` | Each lease needs a person's yes first. |
| `allowFrom` | While a lease is open, only these addresses or CIDRs. It only narrows the service's own `allowFrom`; it never makes it wider. If the two don't overlap, nobody can reach the port. |

- `lease` needs `expose: hostPort` or `expose: loadBalancer`. On a private service it is refused.
- Durations are Go durations (`90s`, `30m`, `2h`), at least one second.
- A leased port **starts closed**. A port not listed in `lease.ports` is open as usual.

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
| `--port NAME` | The leased port, by the name in `service.ports` (required) |
| `--for DURATION` | How long. Empty: the silo's `lease.default`. More than the max: you get the max, and `kvo` says so |
| `--no-wait` | With `kvo open`: if the lease needs a yes, print what to approve and return |

Opening needs the verb `open` on `silos/exposure` ([below](#who-may-do-what)).

### From inside the silo

The silo can ask for its own port, over its [broker]({{ '/secrets/#the-broker-for-everything-else' | relative_url }}):

```bash
kvo-guest port open debug --for 1h
```

```bash
kvo-guest port close debug
```

| Flag | Meaning |
|---|---|
| `--for DURATION` | How long, as above |
| `--wait DURATION` | If it needs a yes: how long to wait for one. Default 10 minutes, at most 1 hour |

`kvo-guest port open` waits for the answer and prints `port debug open until …`. It fails if
the answer is no, or if nobody answers in time. A silo can only open and close **its own leased
ports**: the silo that asks is known from the channel it asks on.

---

## A person's yes

With `approval.required: true`, an ask doesn't open the port. It becomes an
[approval]({{ '/secrets/#ask-a-person-first-approvals' | relative_url }}) with the use `expose`, and the port stays closed:

```bash
kvo open silo node-1 --port debug --for 1h
```

```text
silo/node-1 port debug: waiting for a yes: kvo approve node-1-debug-3f9a1c -n infra (expires 4:12PM)
```

Someone else answers:

```bash
kvo get approvals -n infra
```

```bash
kvo approve node-1-debug-3f9a1c -n infra
```

```bash
kvo deny node-1-debug-3f9a1c -n infra --reason "not today"
```

- **Only people answer**, with the verb `approve` on `silos/exposure`. A silo's own yes is never
  taken.
- **Someone who holds `approve` there, and asks, needs nobody's yes.** Their own ask opens at once.
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
- On a [load balancer from a plugin]({{ '/extending/plugins/' | relative_url }}), the port is taken out of what the plugin is asked to serve, and that is reconsidered at the end of the lease.

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
node-1   rpc     Closed   time up                alice (yes: carol)
```

| Phase | Means |
|---|---|
| `Open` | Open until the time shown. An extension may be waiting |
| `Pending` | An ask waits for a yes; the port is closed |
| `Closed` | Time was up, or someone closed it |
| `Denied` | The last ask was denied |
| `Expired` | The last ask's approval wasn't answered in time |

`kvo describe silo node-1` has a "Leased ports" table with each port's state. There is one
`Lease` per leased port, named `SILO-PORT-xxxxxx`. Only the control plane writes them; you can't
create or edit one, and an admin can delete one.

Every step is in the [audit log]({{ '/operations/#the-audit-log' | relative_url }}) with the
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
see [People and access]({{ '/access/' | relative_url }}#roles).

Making a port public in the first place is still its own permission, `services/expose-public`:
a lease only decides *when*.

---

## Load balancers from a plugin, and `allowFrom`

A lease can narrow who gets in (`lease.allowFrom`), and so can the service's own `allowFrom`.
Some load balancers can't filter by client address. The rule:

| Class | What happens to a port that has sources |
|---|---|
| Announces its address (a plugin class with `announce`: the address is routed to the silo's host, which sees the client's own address) | The host's firewall enforces it |
| A plugin's node-port class with `filtersSources: true` | The plugin is given each port's sources (`allowFrom`) and must enforce them |
| A plugin's node-port class without it | **The port is left out.** The silo's `LoadBalancer` condition says `SourcesNotEnforced` and names the ports |

The last row closes a hole rather than leaving one: a port someone narrowed is never served to
everyone. If you hit it, drop the `allowFrom`, use a host port or an announced class, or use a
plugin that filters. Plugin authors: see [Plugins]({{ '/extending/plugins/' | relative_url }}#load-balancer-classes-and-sources).

---

## Not done

- A lease on an **Ingress** host. Only `hostPort` and `loadBalancer` services lease.
- Leases on a **NodeBalancer** class haven't run on a real machine: the test token had no NodeBalancer access.

See [What's proven where]({{ '/operations/#whats-proven-where' | relative_url }}).
