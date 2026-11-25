---
title: kvo commands
permalink: /reference/kvo/
audience: user
---

# kvo commands

Every `kvo` command on one page. `kvo` uses your current context (`kvo ctx`); `-n NS` picks a
namespace (default: the context's). What each one needs is your role: `kvo auth can-i` tells you
([What you may do]({{ '/use/sign-in/#what-you-may-do' | relative_url }})). A few here are an
admin's, such as namespaces, hosts and IPs; the cluster admin runs them as `kvoadm kvo …`.

The administrator's commands are on [kvoadm and kevasod commands]({{ '/reference/kvoadm/' | relative_url }}).

---

## Objects

| Command | Does |
|---|---|
| `kvo apply -f FILE` (or `-f -`) | Create or update every object in a YAML/JSON file. A field the kind doesn't have is an error, never dropped: fix the name and apply again |
| `kvo get KIND [NAME] [-n NS \| -A] [-o yaml\|json]` | List or show objects |
| `kvo describe KIND NAME` | Everything about one object, for people |
| `kvo delete KIND NAME` | Delete it. A namespace is refused while it still holds anything, and while a binding names the group `system:serviceaccounts:NS`: delete those first. Deleting a namespace never deletes what is in it. `DELETE` on a `/status` path is refused (405) |
| `kvo delete host NAME --confirm-off` | Delete a Host that still has silos, confirming the machine is off or wiped. Refused without the flag. [Removing a host]({{ '/admin/workers/host-loss/#removing-a-host' | relative_url }}) |
| `kvo watch KIND [-n NS \| -A]` | Print the current state, then every change |

Kinds: `namespaces` (`ns`), `hosts`, `silos`, `silosets` (`ss`), `services` (`svc`), `vmimages`
(`vmi`), `secrets`, `grants`, `openshellpolicies`, `ips`, `ingresses` (`ing`), `ingressclasses`, `certificates`
(`cert`), `issuers`, `roles`, `rolebindings`, `clusterroles`, `clusterrolebindings`, `clustersecrets`,
`serviceaccounts` (`sa`), `approvals`, `panes`, `kinddefinitions` (`kd`), `validationpolicies`,
`mutationpolicies`, `leases`, `plugins`, and any [kind you define]({{ '/admin/extending/kinds/' | relative_url }}) (by its plural,
or `PLURAL.GROUP` when two share one).

| Command | Does |
|---|---|
| `kvo get kinds` | Every kind the cluster serves, built-in and defined, with its group and version |
| `kvo get KIND -o json` | Always a JSON array, whether the list has none, one or many |

`kvo apply` says `unchanged` when the server would store the same object it already has, defaults
and policies included. The server does this with a dry run, `?dryRun=All` on a create or update: the full
write, answered with what would be stored, and nothing stored
([Policies]({{ '/admin/extending/policies/#dry-runs' | relative_url }})).

`kvo get`, `describe` and `watch` remove terminal control characters (escape sequences and text-direction
overrides) from status text that silos and hosts write, so a silo can't change what your terminal shows.

## Silos

| Command | Does |
|---|---|
| `kvo exec silo NAME -- CMD [ARGS…]` | Run a command inside; exits with its code; stdin passes through |
| `kvo exec -it silo NAME -- CMD [ARGS…]` | The same on a terminal (`-t`; `-i` is accepted and changes nothing). [Terminals]({{ '/use/silos/terminals/' | relative_url }}) |
| `kvo ssh silo NAME` | A login shell on a terminal (`bash -l` or `sh -l`). Not SSH: the same channel as `exec` |
| `kvo cp SRC DST [--max-size SIZE]` | Copy a file or folder in or out. One side is `silo/NAME:/ABSOLUTE/PATH`. Default limit `1GiB` |
| `kvo ui silo NAME [--no-browser]` | A web page on `127.0.0.1` with the silo's terminal and files. Ctrl-C stops it |
| `kvo reset silo NAME` | Back onto its image: stopped, a fresh disk or container, volumes kept (a set's instance: onto the set's current image) |
| `kvo protect silo NAME [--off]` | A set's scale-in never removes it |
| `kvo move silo NAME --to HOST [--recreate] [--confirm-fenced [--insist]] [-n NS]` | Move it, with its data when its storage allows. A container's own files are never kept: a plain move is refused, and a fresh copy needs `onHostLoss: Recreate` and `--recreate`. `--recreate` is only for `onHostLoss: Recreate` silos. A plain move needs `create` on `silos/move`. `--confirm-fenced`: its host is lost and you know the old copy is off; without it a move from a lost host is refused. Refused while that host claimed a start of this silo less than 30 seconds ago; `--insist`, with `--confirm-fenced`, overrides that. Both flags need `create` on `hosts/confirm-off` (cluster-scoped; only `cluster-admin` by default). [Moves]({{ '/use/silos/moves/' | relative_url }}), [Moving a silo off a host you know is off]({{ '/admin/control-plane-loss/#moving-a-silo-off-a-host-you-know-is-off' | relative_url }}) |
| `kvo get silos [-o wide]` | `NAME`, `RUNTIME`, `PHASE`, `READY`, `HOST`, `AGE`. `-o wide` adds `MOVABLE` (where each silo could go) and `ON-CP-LOSS` (`stops after 2m` for a silo with `onControlPlaneLoss: Stop` and no `after` of its own, or `stops after` that `after`; `-` for the others). A silo on a silent host has phase `Unknown` |
| `kvo describe silo NAME` | Its state, [probes]({{ '/use/silos/probes/' | relative_url }}), what it starts from, what it does when its host loses the control plane (`On control-plane loss`), its conditions (`HostUnreachable`, `FenceArmed`, `FenceStopped`), and each port with its addresses and lease. [Addresses, ports and names]({{ '/use/networking/model/' | relative_url }}) |
| `kvo open silo NAME --port PORT [--port PORT]… [--for D] [--no-wait]` | Open [leased ports]({{ '/use/networking/leases/' | relative_url }}) for a while. Each port is its own lease, on its own terms. If a lease needs a yes, wait for it (`--no-wait`: don't). A refused port doesn't stop the others, and the command exits non-zero |
| `kvo close silo NAME --port PORT [--port PORT]…` | Close them early |
| `kvo get leases [-n NS \| -A]` | The leased ports and their state: `SILO`, `PORT`, `PHASE`, `WHEN`, `BY` |
| `kvo restore status` | The same four lines as `kvoadm restore status`. [Backup and restore]({{ '/admin/operations/backup/#3--wait-until-each-silo-is-settled' | relative_url }}) |
| `kvo restore settle SILO --to HOST [-n NS]` | The same write as `kvoadm restore settle`. Cluster-admin only |

## Panes

| Command | Does |
|---|---|
| `kvo get panes [-n NS \| -A]` | The panes silos report: `SILO`, `PANE`, `KIND`, `STATE`, `UPDATED` |
| `kvo pane watch SILO` | Print a silo's panes, then every change, as plain text |
| `kvo pane bundle SILO.PANE -o FILE` | Save an HTML pane's page to a file (never printed) |

See [Panes]({{ '/use/panes/' | relative_url }}).

## Sets

| Command | Does |
|---|---|
| `kvo scale siloset NAME --instances N` | Grow or shrink; shrinking lists the instances it deletes and needs `--yes` |
| `kvo reset siloset NAME --rolling` | Reset every instance onto the current image, a few at a time |
| `kvo scale KIND NAME --replicas N` | The same for a [defined kind]({{ '/admin/extending/kinds/#scale' | relative_url }}) that declares a scale subresource (`--replicas` and `--instances` are one flag) |

## Secrets

| Command | Does |
|---|---|
| `kvo create secret NAME --from-file F\|-` | Store a value. `--from-literal VALUE` is refused unless you add `--i-know`: the value is on the command line |
| `kvo create secret NAME --generate [--type ed25519\|ecdsa-p256]` | Let Kevaso make it |
| …`--allow MODES`, `--write-only`, `-l KEY=VALUE` | Restrict delivery (`broker`, `inject`, `file`, `env`, comma-separated), make write-only, label |
| `kvo create secret NAME --type registry --server HOST --username U --from-file - [--default-pull]` | A [private registry's]({{ '/use/secrets/' | relative_url }}) credential: write-only, the password on stdin. `--default-pull`: the one used for that registry when a silo names none |
| `kvo secret set NAME --from-file F\|-` | A new value (rotation) |
| `kvo get secret NAME --reveal` | Print the value (needs `read-value`; never for write-only) |
| `kvo create clustersecret NAME [--description TEXT] --from-file F\|-` | A secret with no namespace, for a plugin. Cluster-admin only. The value comes from a file or from stdin. `--from-literal` is refused |
| `kvo set clustersecret NAME --from-file F\|-` | Replace that value. Each host that has it writes the new copy and restarts the plugin |
| `kvo get clustersecret [NAME] [--reveal]` | The name and description. `--reveal` prints the value. A value is readable by a cluster admin, or by `get` on `clustersecrets/value` in a ClusterRole whose `resourceNames` lists that name |
| `kvo delete clustersecret NAME` | Delete it |
| `kvo get approvals` / `kvo watch approvals` | Silos waiting for a person's yes. The `REQUEST` column is 32 hex digits, the start of the request's 64-digit id |
| `kvo approve NAME [--request ID]` | Say yes (needs `approve` on that secret; people only). `--request` is that column, a longer prefix, or the full 64 hex digits. At least 32 digits. On a terminal you can omit it: the request is shown and the yes is for what was shown. [Approvals]({{ '/use/secrets/#ask-a-person-first-approvals' | relative_url }}) |
| `kvo deny NAME [--reason R]` | Say no |

## Addresses you keep

| Command | Does |
|---|---|
| `kvo create ip NAME --class CLASS [--host HOST \| --region REGION] [--address A] [--reclaim Retain\|Delete]` | Reserve an address. `--reclaim` defaults to `Retain`. [Addresses you keep]({{ '/admin/ips/' | relative_url }}) |
| `kvo get ips` | `ADDRESS`, `CLASS`, `HELD-BY`, `KEPT` |
| `kvo describe ip NAME` | What holds it and on which ports |
| `kvo ip keep NAME` | Its address stays when its holder goes (and it is no longer owned by it) |
| `kvo ip release NAME` | It goes with its holder, or now if nothing holds it |

## Silo images (microVMs)

| Command | Does |
|---|---|
| `kvo get vmimages` | The disk images microVM silos can start from: `VERSIONS`, `DISK` |

## Plugins for `kvo`

| Command | Does |
|---|---|
| `kvo plugin search [TERM]` | The plugins every index offers, and whether there is a build for this machine |
| `kvo plugin install NAME[@VERSION]` | Install the build for this machine into your own folder, after checking its signature and hash |
| `kvo plugin list` | What is installed, and from which index |
| `kvo plugin upgrade [NAME]` | To the newest release (every plugin without `NAME`) |
| `kvo plugin remove NAME` | Remove it |
| `kvo plugin index add URL --key ed25519:… [--name N]` | Trust another publisher's index; its plugins are `N/NAME` |
| `kvo plugin index list` / `remove N` | See or drop indexes |
| `kvo plugin token [--context NAME]` | For a plugin to call: a short-lived access token, as JSON |
| `kvo NAME [ARGS…]` | Run an installed plugin in the current context |

A plugin never takes the name of a `kvo` command. See [Plugins for kvo]({{ '/use/kvo-plugins/' | relative_url }}).

## Access

| Command | Does |
|---|---|
| `kvo login SERVER --ca sha256:…` | Sign in (browser + invite code). `SERVER` is `https://` only. Several control planes from an invite are comma-separated: each is tried for the CA, and the whole list is kept in the context |
| `kvo invite NAME [--group G]… [-n NS --role role/NAME\|clusterrole/NAME \| --cluster-role NAME] [--ttl D]` | Bring a person in: prints the `kvo login SERVER --ca sha256:…` line and a one-time code to send them. Cluster-admin only, however you signed in. `-n NS --role role/NAME`: the Role `NAME` in `NS`. `-n NS --role clusterrole/NAME`: the ClusterRole `NAME`, in `NS` only. `--cluster-role NAME`: the ClusterRole in every namespace. No role flag: no role. A role without its kind, `--role` without `-n`, `--cluster-role` with `-n`, both, or a missing role is refused before anything is made. `--ttl` defaults to `15m`, at most `168h`. Releases after `v0.1.0-rc.8`. [Invite someone]({{ '/admin/access/#invite-someone' | relative_url }}) |
| `kvo logout` | Forget this context's session |
| `kvo ctx [NAME]` | List contexts, or switch |
| `kvo ns [NAME]` | Show or set the default namespace |
| `kvo auth whoami` | Who you are to the cluster |
| `kvo auth can-i VERB RESOURCE[/NAME] [-n NS]` | Exits 0 for yes, 1 for no. `ssh`, `files`, `open`, `close` and `move` are silo subresources (`silos/ssh` is the terminal, not a silo named `ssh`) |
| `kvo create token SA [--duration 1h]` | A service-account token, for CI (`$KVO_TOKEN`). Default one hour, at most 2160h (90 days). A namespace editor can't mint one |

## Environment

| Variable | Means |
|---|---|
| `KVO_CONFIG` | The config file (default `~/.kevaso/config.yaml`). `kvoadm` doesn't read it |
| `KVO_TOKEN`, `KVO_CA_FILE` | A service-account token and the cluster CA, for CI |
| `KVO_SERVER` | A plain-HTTP server (`kevasod dev`, local experiments) |
