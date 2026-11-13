---
title: High availability
permalink: /operations/ha/
---

# High availability: more than one system node

One system node is one point of failure. With **three system nodes** (or two machines plus a small
third vote), you can lose one machine and nothing stops: the API answers, silos are scheduled,
`kvo exec` works.

This page is about the machines that run the control plane. For what happens to the *silos* on a
lost worker, see [Host loss and moves]({{ '/silos/moves/' | relative_url }}).

---

## What you get

| Setup | Machines | Lose one |
|---|---|---|
| One system node (default) | 1 | The control plane is down until it's back |
| Three system nodes | 3 | Nothing stops |
| Two system nodes and a **light node** | 3 (one small) | Nothing stops |
| Two system nodes only | 2 | **Don't.** Losing either stops the store: two voters need both |

Every system node serves the API. One of them, the **leader**, also runs the controllers and the
scheduler. The others are ready to take over.

A **light node** (`--state-only`) runs only a vote in the store: no API, no controllers, no mesh,
no silos. It exists so two big machines and one tiny one make three votes. It can be the smallest
machine you have.

---

## Start on etcd

HA needs [the etcd store]({{ '/operations/#the-store' | relative_url }}). Start a new cluster on it:

```bash
kvoadm init root@203.0.113.10 --ssh-key ~/.ssh/kevaso_ed25519 --kevasod ./kevasod --store etcd
```

This also lets the first node listen for other members (port 2380).

A cluster that already runs on SQLite is switched with one command, once:

```bash
kvoadm store migrate root@203.0.113.10 --ssh-key ~/.ssh/kevaso_ed25519
```

It rewrites the control plane's service to `--store etcd` with a peer listener, and restarts it.
The API is away for a moment, and every `kvo` and worker re-lists once, by itself. Silos keep
running. [Take a backup first]({{ '/operations/backup/' | relative_url }}): there's no switch back.

`kvoadm host add … --install control-plane` refuses a SQLite cluster and prints this command.

---

## Add a system node

On a fresh machine you can reach over SSH:

```bash
kvoadm host add root@203.0.113.11 --ssh-key ~/.ssh/kevaso_ed25519 --kevasod ./kevasod --install control-plane --name sys2
```

```bash
kvoadm host add root@203.0.113.12 --ssh-key ~/.ssh/kevaso_ed25519 --kevasod ./kevasod --install control-plane --name sys3
```

One at a time: a second join waits for the first. What happens:

1. The host is declared and a one-time join token is made. It's good for the role `system-node`
   only, and it's spent when the join succeeds.
2. The new node starts `kevasod control-plane --join …`, and asks to join with the token.
3. The cluster adds it to the store as a **learner**, and sends it the cluster's CA and the master key,
   sealed so that only that node can open them. `kvoadm` never sees them.
4. When it has caught up, the leader promotes it to a voter. `kvoadm` waits for that (up to five
   minutes), prints the API address, and adds it to your `kvo` context.

| Flag | Meaning |
|---|---|
| `--install control-plane` | A system node, not a worker |
| `--state-only` | A light node: a vote only |
| `--cluster-dns NAME` | A DNS name with one record per system node. It goes in every node's certificate and is the one sign-in address, so tokens work at any node |
| `--address ADDR` | The address other nodes reach it at, if not the SSH one |

Open **TCP 2380 between the system nodes only**. The members talk over the host's address,
not the mesh, with mutual TLS: only a member's certificate is let in.

### An existing worker becomes a system node too

```bash
kvoadm host install hv2 control-plane --ssh-key ~/.ssh/kevaso_ed25519
```

It keeps its worker part. (`--state-only` here would drop the worker part: give a light node a
machine of its own.)

### The light node, for two machines

Two machines plus the small one:

```bash
kvoadm host add root@203.0.113.12 --ssh-key ~/.ssh/kevaso_ed25519 --kevasod ./kevasod --install control-plane --state-only --name vote
```

See them:

```bash
kvoadm host list
```

```bash
kvo describe host sys2
```

`describe` shows `status.controlPlane`: whether it's a `learner` or `voter`, and which one leads.

---

## Losing a node

| What | What happens |
|---|---|
| **A system node** (of three) | The others keep a majority. The API answers at the others. If the lost one led, another takes over after **15 seconds** of silence |
| **The light node** | Two voters are left: nothing stops, but now losing either of those stops the store. Fix it soon |
| **Two of three** | The store has no majority and stops taking changes. Silos keep running. Bring one back |
| **It comes back** | It catches up from the others by itself |

### What `kvo` does

A `kvo` context can list several servers. It sticks to the last one that answered, and moves to the
next only when it **couldn't connect** at all (so nothing is ever sent twice). A dead machine costs
a few seconds, not a long hang: the connect gives up after 5 seconds.

`kvoadm host add` adds each new node to your context. On another computer, list them yourself:
the server is comma-separated, in `~/.kevaso/config.yaml`.

### What workers do

A worker keeps a connection to **every** system node, and remembers them (in
`/var/lib/kevaso/pki/servers`), including a worker that was added when there was only one control
plane. When a system node is lost, the worker uses the others for its heartbeat and its watches,
and `kvo exec` reaches its silos through any node.

### Not yet covered

- A **system node that is also a worker** (`kvoadm host install … control-plane` on a worker, or
  `kvoadm init --worker`) holds one connection, to its own node. A `kvo exec` into its silos
  that lands on another system node fails until that node is back.
- Silo services run inside `kevasod`: the silo DNS, the egress gate, the secret broker and open
  `exec` streams. They blip for the seconds a restart takes.
- **Secrets' master key rotation is refused** while there is more than one member: each node
  keeps its own copy of the key.
- A person's sign-in is two steps, and they must reach the same node. `kvo` does that by itself.

---

## Remove a system node

```bash
kvoadm host remove sys3
```

The node leaves the store first, and the Host is deleted after. It's **refused** when removing it
would lose the store's healthy majority, or leave no node serving the API. Fix or add another node
first, then retry.

The machine itself is yours afterwards; stop it with `systemctl disable --now kevasod`.

If the host is a worker too, take off only the control plane:

```bash
kvoadm host uninstall hv2 control-plane
```

On a host that runs only the control plane, `uninstall` is refused (a host needs at least one
component): use `remove`.

---

## Upgrade

```bash
kvoadm upgrade --kevasod ./kevasod --dry-run
```

It prints the plan and changes nothing. Without `--dry-run` it does it:

```bash
kvoadm upgrade --kevasod ./kevasod
```

| Flag | Meaning |
|---|---|
| `--kevasod PATH` | The new `kevasod`, for the hosts' CPU |
| `--version V` | Its version, if the build doesn't carry one |
| `--batch N` | How many workers restart at once (default 2) |
| `--ssh-key PATH` | The key for the hosts |
| `--context NAME` | The `kvo` context to act as |
| `--dry-run` | Only the plan |

**The order:** light nodes first, the other system nodes next, the leader last (so there's one
handover); then workers, a few at a time. It signs the binary as `kvoadm host add` does; each host
checks the signature against the key it was set up with before it swaps the program in, and
restarts it. Then it waits until the host is `Ready` and reports the new version before it goes on.
It stops at the first failure and says what's left.

**Silos keep running** while their host's `kevasod` restarts: a microVM is its own service, a
container belongs to Docker or Podman. Docker is told to keep containers alive when its daemon
restarts (`live-restore`), and Kevaso restarts Docker only when its own settings change.

**It refuses** when:

- Restarting one more system node would lose the store's majority (`2 of 3 system nodes are
  Ready`…). With two system nodes and no third vote it tells you to add a light node first.
- A host reports no version, or the move breaks the rule below.

With one system node it goes ahead and says the API is away while it restarts.

### The version rule

| Rule | |
|---|---|
| An upgrade only moves **forward**, by **at most one minor version**, inside one major | `v0.4.2` → `v0.5.0`: yes. `v0.4.2` → `v0.6.0`: no, go through `v0.5` |
| System nodes may be **at most one minor apart** | During the roll only |
| A worker is **never newer** than the oldest system node, and at most one minor older | So system nodes always go first |

A cluster already outside the rule is refused with the reason. Each host shows its version in
`kvo get hosts -o yaml` (`status.version`).

Upgrade by hand if a host is older than version reporting. Self-update (a host pulling a signed
version itself) is not built.

---

## What isn't proven

On real machines this has run on Linodes in one setup: three system nodes, and two with a light
node, losing one, removing one, upgrading, and SQLite to etcd. **Not tried:** removing the
leader's own node, five system nodes, an upgrade with a light node in the cluster, Podman workers.
See [What's proven where]({{ '/operations/#whats-proven-where' | relative_url }}).

Not built: warnings when a member is slow, a primary with a standby, a shared master key, etcd
defragmenting on a schedule.
