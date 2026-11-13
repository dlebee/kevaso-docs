---
title: Plugins
permalink: /extending/plugins/
---

# Plugins

A plugin adds to Kevaso from outside. There are two sorts, and `kvoadm plugin` installs both:

| Sort | Adds | Example |
|---|---|---|
| **Provider plugin** | A cloud or a storage system: load balancers, block volumes, a shared file system, DNS | [Linode]({{ '/providers/linode/' | relative_url }}), [JuiceFS]({{ '/providers/juicefs/' | relative_url }}) |
| **Plugin operator** | New [kinds]({{ '/extending/kinds/' | relative_url }}), their [policies]({{ '/extending/policies/' | relative_url }}), and a controller that acts on them | [The example below](#the-example-sessions) |

Either way, a plugin never gets more than it shows you first.

---

## Install a provider plugin

```bash
kvoadm plugin install root@203.0.113.10 --name linode --binary ./kevaso-provider-linode-linux-amd64 --credential token=$HOME/linode-token.txt
```

| Flag | Meaning |
|---|---|
| `--name NAME` | What to call it (lower case, digits and `-`) |
| `--binary FILE` | The plugin's program, built for the host. `kvoadm` signs it; the host checks the signature before it installs it |
| `--credential NAME=FILE` | A secret, read from a file (never the command line), kept on the host root-only and handed to the plugin through systemd |
| `--config FILE` | Its settings (not secret) |
| `--side controller\|node` | `controller` on a control plane (default), `node` on a worker |
| `--manifest FILE` | What the plugin needs from the host: [below](#what-a-manifest-asks-for) |
| `--yes` | Grant the privileges the manifest asks for |

To remove one:

```bash
kvoadm plugin remove root@203.0.113.10 --name linode
```

### The default is small

Without a manifest, a plugin runs as its own throwaway user, with no privileges, no devices, a
read-only system, its own `/tmp`, and only the network and its own socket. That is enough for
a plugin that calls a cloud's API.

### What a manifest asks for

A plugin that has to mount file systems can't live in that. Its package carries a
`kevaso-plugin.json` that says what **each side** needs:

```json
{
  "name": "juicefs",
  "controller": { "root": true, "capabilities": ["CAP_SYS_ADMIN"], "devices": ["/dev/fuse"], "writePaths": ["/var/lib/kevaso-provider-juicefs"] },
  "node": { "root": true, "capabilities": ["CAP_SYS_ADMIN"], "devices": ["/dev/fuse"], "hostMounts": true }
}
```

| Field | Gives the side |
|---|---|
| `root` | Runs as root, not a throwaway user |
| `capabilities` | Exactly these, and no others (like `CAP_SYS_ADMIN` to mount) |
| `devices` | Read-write access to these devices, and no others (like `/dev/fuse`) |
| `hostMounts` | The host's own mount namespace, so what it mounts is seen by `kevasod`, the container runtime and `virtiofsd`. Needs `root`. **It costs the read-only system and the private `/tmp`**, since those need a namespace of their own |
| `writePaths` | Folders it may write; the rest of the system stays read-only. Not with `hostMounts` |
| `socket` | `plugin.sock` or `node.sock`: the only two `kevasod` looks for |

A manifest naming something `kvoadm` doesn't know is refused, so a need is never dropped silently.

**`kvoadm` shows you what you'd grant, and refuses to go on without `--yes`:**

```bash
kvoadm plugin install root@203.0.113.20 --name juicefs --side node --binary ./kevaso-provider-juicefs-linux-amd64 --manifest ./kevaso-plugin.json
```

```text
→ plugin juicefs, node side:
    runs as root (uid 0)
    capabilities: CAP_SYS_ADMIN (all others dropped)
    devices: /dev/fuse (no others)
    the host's mount namespace: its mounts are the host's, and the host's files aren't read-only to it
kvoadm: plugin juicefs's node side asks for the privileges above (manifest ./kevaso-plugin.json): add --yes to grant them
```

A side that asks for nothing extra installs without `--yes`. Everything else a unit has
(a system-call filter, no new privileges, no realtime, …) stays on.

Read it before you say yes: root with `CAP_SYS_ADMIN` and the host's mount namespace is a lot to
give a program, and this one handles your storage.

> **Not proven on a real machine:** a privileged node side installed this way hasn't run on a real
> host yet.

### Load balancer classes and sources

A plugin that provides load balancers lists **classes**. Two settings matter for who may connect:

| Class setting | Means |
|---|---|
| `announce` | Its address is routed to whichever host has the silo. Clients reach that host with their own address, so **the host's firewall** applies `allowFrom` |
| `filtersSources` | The load balancer itself accepts clients only from the sources it's given: the plugin gets each port's `allowFrom` (the service's, narrowed by an open [lease]({{ '/networking/leases/' | relative_url }})) and **must apply it**, say with a firewall on the load balancer |

A node-port class that does neither **gets no port that has sources**: the silo's `LoadBalancer`
condition is `SourcesNotEnforced` and names the ports. A port that someone narrowed is never
quietly opened to everyone. A plugin sets `filtersSources: true` once it applies the sources it's given.

---

## Plugin operators

A plugin operator brings kinds of its own and a controller for them. You install it from a
**package**: a folder of plain files you can read.

```text
sessions/
  plugin.yaml          the package: name, version, the controller, the files below
  kinds.yaml           KindDefinitions
  policies.yaml        ValidationPolicies and MutationPolicies, on the package's own kinds
  roles.yaml           what the controller may do
  bin/kevaso-operator-session     the controller, built for the host
```

`plugin.yaml`:

```yaml
apiVersion: kevaso/v1
kind: PluginPackage
metadata:
  name: sessions
spec:
  version: 0.1.0
  description: Sessions, each a set of silos made from a template
  controller:
    binary: bin/kevaso-operator-session
  resources:
    - kinds.yaml
    - policies.yaml
    - roles.yaml
```

| Field | Meaning |
|---|---|
| `metadata.name` | Lower case, digits and `-`, at most 31 characters. It starts the name of every policy and role in the package |
| `spec.version` | A semantic version (`0.1.0`) |
| `spec.controller.binary` | The controller program: a file in the package, built for the host |
| `spec.resources` | The YAML files of the package. Only `KindDefinition`, `ValidationPolicy`, `MutationPolicy`, `ClusterRole` and `Role` objects. Anything else is refused |

### What a package may not do

`kvoadm` checks all of this **before it writes anything**:

- **Policies reach only the package's own kinds**, by resource (`sessions.example.dev`). Never a
  built-in kind, never `*`. Installing a plugin can't change how the cluster's own objects are
  checked.
- **Roles are small.** Never `*` as a verb or resource. Never RBAC (`roles`, `rolebindings`,
  `clusterroles`, `clusterrolebindings`), never `serviceaccounts/token`, `kinddefinitions`,
  `validationpolicies`, `mutationpolicies` or `plugins`, and never `read-value` or `approve` on
  secrets. The controller can't make itself more than its package says, and is never
  `cluster-admin`.
- **Names say whose they are.** Policies and roles are named `PLUGIN-…`.

### Install

Build the controller, and install:

```bash
kvoadm plugin install root@203.0.113.10 --package ./sessions
```

It prints **every rule the controller will have**, each role and where it applies, and what
every controller gets to run at all (read its own `Plugin`, report its heartbeat, renew its own
token). It installs only when you type `yes`.

| Flag | Meaning |
|---|---|
| `--package DIR` | The package folder |
| `--binary FILE` | The controller, if not the one the package names |
| `--api-server URL` | How the host reaches the API, if that isn't how your computer does |
| `--adopt` | Take over objects of the same names that no plugin owns |
| `--yes` | Don't ask |
| `--ssh-key`, `--host-key`, `--ssh-port` | As for [other `kvoadm` commands]({{ '/reference/' | relative_url }}) |
| `--context NAME` | The `kvo` context to act as (a cluster admin) |

What it does:

1. Makes a service account, `kevaso-system/plugin-NAME`, bound to **exactly** the package's roles.
2. Makes the kinds, policies and roles. Each is owned by a `Plugin` object named after the package.
   Install decides object by object: missing, create; this plugin's, update (an upgrade); another
   plugin's, refused; nobody's, refused unless `--adopt`. An upgrade deletes what the new
   version no longer has.
3. Gives the host an **hour-long token**, with the API address and the cluster CA, in
   `/etc/kevaso/plugins/NAME/`, readable by root only, and runs the controller as a service
   with its own throwaway user, the same sandbox as provider plugins, and its credentials
   through systemd (never on a command line or in its environment).
4. The controller **renews the token itself**, at half its life. If it is down longer than a
   token lasts, install again: that gives it a new one.

Deleting the service account ends every token it has.

### List and uninstall

```bash
kvoadm plugin list
```

```text
NAME      VERSION  KINDS                                         CONTROLLER  HOST
sessions  0.1.0    sessions.example.dev,sessiontemplates.example.dev  running     root@203.0.113.10
```

`running` means a heartbeat in the last 90 seconds. Otherwise it says `not running` and when it was
last seen.

```bash
kvoadm plugin uninstall sessions
```

**Uninstall is refused while objects of its kinds exist**, and tells you how many. Delete them, or:

```bash
kvoadm plugin uninstall sessions --force
```

`--force` deletes the objects (waiting up to 30 seconds for a running controller to clean up),
stops and removes the controller's service, program and credentials, and deletes everything the
plugin owns. Run again, it finishes an uninstall that stopped half-way. While a plugin is
installed, its kinds can't be deleted on their own.

### The example: sessions

Core's repository has a complete one in `examples/operator-session`: two kinds, `SessionTemplate`
and `Session`, and a controller that gives each session `spec.replicas` silos made from its
template, named `SESSION-0`, `SESSION-1`…. Its README walks through it. The controller is Go, on
`kevaso.com/core/api` and `kevaso.com/core/sdk` only.

```bash
kvoadm plugin install root@203.0.113.10 --package examples/operator-session
```

```bash
kvo apply -f examples/operator-session/sample.yaml -n dev
```

```bash
kvo get sessions -n dev
```

```bash
kvo scale session fix-login-bug --replicas 2 -n dev
```

### Writing a controller

Use `sdk/operator` on top of `sdk/controller` ([Defining your own kind]({{ '/extending/kinds/#a-controller' | relative_url }})):

| | |
|---|---|
| `operator.FromEnv()` | The credentials the install put in the service's environment |
| `creds.Client(ctx)` | An API client that signs in with the token and renews it |
| `operator.Heartbeat(ctx, c, name, version)` | Reports every 30 seconds, so `plugin list` can say `running` |

### Scale and transition rules

Two features of [defined kinds]({{ '/extending/kinds/' | relative_url }}) that operators use:

- **A scale subresource**, so `kvo scale` works on your kind:
  `subresources: { scale: { specReplicasPath: .spec.replicas, statusReplicasPath: .status.replicas } }`.
- **Transition rules** with `oldSelf` in `x-kevaso-validations`: `self == oldSelf` (it can't change),
  `self >= oldSelf` (it only grows). They run on updates only.

### What's not here

- **Signed packages.** The program is signed for the host, but a package isn't signed as a whole.
- A controller in a **container or a silo**, or more than one copy with leader election.
- Cleaning up what a plugin made by owner references (the example uses a finalizer for its silos).
- Installing the controller on every control-plane host.

**On real machines:** installing an operator over SSH and systemd, and uninstalling it, has run on
Linodes.
