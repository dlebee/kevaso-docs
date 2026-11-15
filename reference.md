---
title: Command reference
permalink: /reference/
---

# Command reference

Every command on one page. `kvo` uses your current context (`kvo ctx`); `-n NS` picks a namespace
(default: the context's).

---

## kvo: using the platform

### Objects

| Command | Does |
|---|---|
| `kvo apply -f FILE` (or `-f -`) | Create or update every object in a YAML/JSON file. A field the kind doesn't have is an error, never dropped: fix the name and apply again |
| `kvo get KIND [NAME] [-n NS \| -A] [-o yaml\|json]` | List or show objects |
| `kvo describe KIND NAME` | Everything about one object, for people |
| `kvo delete KIND NAME` | Delete it |
| `kvo watch KIND [-n NS \| -A]` | Print the current state, then every change |

Kinds: `namespaces` (`ns`), `hosts`, `silos`, `silosets` (`ss`), `services` (`svc`), `vmimages`
(`vmi`), `secrets`, `grants`, `ips`, `ingresses` (`ing`), `ingressclasses`, `certificates`
(`cert`), `issuers`, `roles`, `rolebindings`, `clusterroles`, `clusterrolebindings`,
`serviceaccounts` (`sa`), `approvals`, `panes`, `kinddefinitions` (`kd`), `validationpolicies`,
`mutationpolicies`, `leases`, `plugins`, and any [kind you define]({{ '/extending/kinds/' | relative_url }}) (by its plural,
or `PLURAL.GROUP` when two share one).

| Command | Does |
|---|---|
| `kvo get kinds` | Every kind the cluster serves, built-in and defined, with its group and version |
| `kvo get KIND -o json` | Always a JSON array, whether the list has none, one or many |

`kvo apply` says `unchanged` when the server would store the same object it already has, defaults
and policies included. The server does this with a dry run, `?dryRun=All` on a create or update: the full
write, answered with what would be stored, and nothing stored
([Policies]({{ '/extending/policies/#dry-runs' | relative_url }})).

`kvo get`, `describe` and `watch` remove terminal control characters (escape sequences and text-direction
overrides) from status text that silos and hosts write, so a silo can't change what your terminal shows.

### Silos

| Command | Does |
|---|---|
| `kvo exec silo NAME -- CMD [ARGS…]` | Run a command inside; exits with its code; stdin passes through |
| `kvo exec -it silo NAME -- CMD [ARGS…]` | The same on a terminal (`-t`; `-i` is accepted and changes nothing). [Terminals]({{ '/silos/terminals/' | relative_url }}) |
| `kvo ssh silo NAME` | A login shell on a terminal (`bash -l` or `sh -l`). Not SSH: the same channel as `exec` |
| `kvo cp SRC DST [--max-size SIZE]` | Copy a file or folder in or out. One side is `silo/NAME:/ABSOLUTE/PATH`. Default limit `1GiB` |
| `kvo ui silo NAME [--no-browser]` | A web page on `127.0.0.1` with the silo's terminal and files. Ctrl-C stops it |
| `kvo reset silo NAME` | Back onto its image: stopped, a fresh disk or container, volumes kept (a set's instance: onto the set's current image) |
| `kvo protect silo NAME [--off]` | A set's scale-in never removes it |
| `kvo move silo NAME --to HOST [--recreate] [--confirm-fenced] [-n NS]` | Move it with its data. `--recreate`: a fresh copy from its image (only `onHostLoss: Recreate` silos). `--confirm-fenced`: its host is lost and you know the old copy is off; without it a move from a lost host is refused. Refused while that host claimed a start of this silo less than 30 seconds ago. [Moves]({{ '/silos/moves/' | relative_url }}), [Moving a silo off a host you know is off]({{ '/silos/control-plane-loss/#moving-a-silo-off-a-host-you-know-is-off' | relative_url }}) |
| `kvo get silos [-o wide]` | `NAME`, `RUNTIME`, `PHASE`, `READY`, `HOST`, `AGE`. `-o wide` adds `MOVABLE` (where each silo could go) and `ON-CP-LOSS` (`stops after 2m` for a silo with `onControlPlaneLoss: Stop`, `-` for the others). A silo on a silent host has phase `Unknown` |
| `kvo describe silo NAME` | Its state, [probes]({{ '/silos/probes/' | relative_url }}), what it starts from, what it does when its host loses the control plane (`On control-plane loss`), its conditions (`HostUnreachable`, `FenceArmed`), and each port with its addresses and lease. [Addresses, ports and names]({{ '/networking/model/' | relative_url }}) |
| `kvo open silo NAME --port PORT [--port PORT]… [--for D] [--no-wait]` | Open [leased ports]({{ '/networking/leases/' | relative_url }}) for a while. Each port is its own lease, on its own terms. If a lease needs a yes, wait for it (`--no-wait`: don't). A refused port doesn't stop the others, and the command exits non-zero |
| `kvo close silo NAME --port PORT [--port PORT]…` | Close them early |
| `kvo get leases [-n NS \| -A]` | The leased ports and their state: `SILO`, `PORT`, `PHASE`, `WHEN`, `BY` |

### Panes

| Command | Does |
|---|---|
| `kvo get panes [-n NS \| -A]` | The panes silos report: `SILO`, `PANE`, `KIND`, `STATE`, `UPDATED` |
| `kvo pane watch SILO` | Print a silo's panes, then every change, as plain text |
| `kvo pane bundle SILO.PANE -o FILE` | Save an HTML pane's page to a file (never printed) |

See [Panes]({{ '/panes/' | relative_url }}).

### Sets

| Command | Does |
|---|---|
| `kvo scale siloset NAME --instances N` | Grow or shrink; shrinking lists the instances it deletes and needs `--yes` |
| `kvo reset siloset NAME --rolling` | Reset every instance onto the current image, a few at a time |
| `kvo scale KIND NAME --replicas N` | The same for a [defined kind]({{ '/extending/kinds/#scale' | relative_url }}) that declares a scale subresource (`--replicas` and `--instances` are one flag) |

### Secrets

| Command | Does |
|---|---|
| `kvo create secret NAME --from-file F\|-` | Store a value (never from the command line) |
| `kvo create secret NAME --generate [--type ed25519\|ecdsa-p256]` | Let Kevaso make it |
| …`--allow MODES`, `--write-only`, `-l KEY=VALUE` | Restrict delivery (`broker`, `inject`, `file`, `env`, comma-separated), make write-only, label |
| `kvo create secret NAME --type registry --server HOST --username U --from-file - [--default-pull]` | A [private registry's]({{ '/secrets/' | relative_url }}) credential: write-only, the password on stdin. `--default-pull`: the one used for that registry when a silo names none |
| `kvo secret set NAME --from-file F\|-` | A new value (rotation) |
| `kvo get secret NAME --reveal` | Print the value (needs `read-value`; never for write-only) |
| `kvo get approvals` / `kvo watch approvals` | Silos waiting for a person's yes. The `REQUEST` column is 32 hex digits, the start of the request's 64-digit id |
| `kvo approve NAME [--request ID]` | Say yes (needs `approve` on that secret; people only). `--request` is that column, a longer prefix, or the full 64 hex digits. At least 32 digits. On a terminal you can omit it: the request is shown and the yes is for what was shown. [Approvals]({{ '/secrets/#ask-a-person-first-approvals' | relative_url }}) |
| `kvo deny NAME [--reason R]` | Say no |

### Addresses you keep

| Command | Does |
|---|---|
| `kvo create ip NAME --class CLASS [--host HOST \| --region REGION] [--address A] [--reclaim Retain\|Delete]` | Reserve an address. `--reclaim` defaults to `Retain`. [Addresses you keep]({{ '/networking/ips/' | relative_url }}) |
| `kvo get ips` | `ADDRESS`, `CLASS`, `HELD-BY`, `KEPT` |
| `kvo describe ip NAME` | What holds it and on which ports |
| `kvo ip keep NAME` | Its address stays when its holder goes (and it is no longer owned by it) |
| `kvo ip release NAME` | It goes with its holder, or now if nothing holds it |

### Silo images (microVMs)

| Command | Does |
|---|---|
| `kvo get vmimages` | The disk images microVM silos can start from: `VERSIONS`, `DISK` |

### Plugins for `kvo`

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

A plugin never takes the name of a `kvo` command. See [Plugins for kvo]({{ '/extending/kvo-plugins/' | relative_url }}).

### Access

| Command | Does |
|---|---|
| `kvo login SERVER --ca sha256:…` | Sign in (browser + invite code) |
| `kvo logout` | Forget this context's session |
| `kvo ctx [NAME]` | List contexts, or switch |
| `kvo ns [NAME]` | Show or set the default namespace |
| `kvo auth whoami` | Who you are to the cluster |
| `kvo auth can-i VERB RESOURCE[/NAME] [-n NS]` | Exits 0 for yes, 1 for no |
| `kvo create token SA [--duration 1h]` | A service-account token, for CI (`$KVO_TOKEN`) |

### Environment

| Variable | Means |
|---|---|
| `KVO_CONFIG` | The config file (default `~/.kevaso/config.yaml`) |
| `KVO_TOKEN`, `KVO_CA_FILE` | A service-account token and the cluster CA, for CI |
| `KVO_SERVER` | A plain-HTTP server (`kevasod dev`, local experiments) |

---

## kvoadm: running the platform

### Machines

| Command | Does |
|---|---|
| `kvoadm init [USER@]ADDR --ssh-key KEY --host-key SHA256:… --kevasod BIN [--name N] [--cluster C] [--api-server URL] [--store sqlite\|etcd] [--worker] [--storage-device /dev/DISK \| --loop-storage SIZE]` | Create the cluster (the only command that does), install the control plane, write your kvo context, print the recovery key. `--store etcd`: start on etcd, ready for [more system nodes]({{ '/operations/ha/' | relative_url }}). `--worker`: the machine runs silos too. [Starting a cluster]({{ '/operations/backup/#starting-a-cluster-and-a-node-that-finds-its-data-gone' | relative_url }}) |
| `kvoadm host enable-worker [USER@]ADDR --ssh-key KEY [--runtime R]… [--loop-storage SIZE \| --storage-device /dev/DISK]` | The control-plane host runs silos too |
| `kvoadm host reconnect [USER@]ADDR --ssh-key KEY [--server URL]` | Point a worker at the control plane again (after a restore onto another address) |
| `kvoadm host trust-control-plane [USER@]ADDR --ssh-key PATH [--host-key SHA256:…] [--ssh-port N] [--data DIR] [--yes]` | Set aside a worker's unreadable record of its control plane and restart `kevasod`. A readable record is left alone. [An unreadable record]({{ '/silos/control-plane-loss/#an-unreadable-record' | relative_url }}) |
| `kvoadm host add [USER@]ADDR --ssh-key KEY --kevasod BIN --install worker [--name N] [--region R] [--zone Z] [--label K=V] [--runtime R] [--behind-nat] [--internal-ip IP] [--external-ip IP] [--storage-device /dev/DISK \| --loop-storage SIZE]` | Add a worker. `--storage-device`: a whole, empty disk for the silos' volume group, when the host has none. `--loop-storage`: development only, a sparse file instead. With neither and no `kevaso` volume group, the microVM runtime stops with a message. [Where silo disks live]({{ '/workers/#where-silo-disks-live' | relative_url }}) |
| `kvoadm host add [USER@]ADDR … --install control-plane [--state-only] [--cluster-dns NAME] [--internal-ip IP] [--external-ip IP]` | A second or third [system node]({{ '/operations/ha/#add-a-system-node' | relative_url }}); `--state-only`: a light node (a vote only) |
| `kvoadm host install HOST control-plane [--state-only] [--ssh-key KEY]` | Make an existing worker a system node too |
| `kvoadm host uninstall HOST control-plane` | Take the control plane off a host that is a worker too (refused if it would lose the store's majority) |
| `kvoadm host remove HOST` | Remove a system node (or a worker's Host): out of the store first; refused if it would lose the majority |
| `kvoadm store migrate [USER@]ADDR --ssh-key KEY` | Move a SQLite control plane onto etcd with a peer listener |
| `kvoadm upgrade --kevasod BIN [--ssh-key KEY] [--batch N] [--version V] [--dry-run]` | Roll a new `kevasod` over the cluster: system nodes (light first, leader last), then workers. [Rules]({{ '/operations/ha/#upgrade' | relative_url }}) |
| `kvoadm host list` | Hosts, both of their IPs, and their state |
| `kvoadm host addresses HOST add\|remove CIDR…` | The extra addresses routed to a host, which the built-in `ip-pool` class hands out ([Addresses you keep]({{ '/networking/ips/' | relative_url }})). `remove` is refused for an address an IP holds |
| `kvoadm host addresses HOST list` | Each range, and which IP holds each address |
| `kvoadm host drain NAME` | Stop new silos landing on the host, and hand off the silos that can move, the way `kvo move` does. Silos that must stay with their local disk are listed. [Draining a host]({{ '/workers/host-loss/' | relative_url }}) |
| `kvoadm host uncordon NAME` | The host takes new silos again |
| `kvoadm reset [USER@]ADDR --ssh-key KEY [--host-key SHA256:…] [--ssh-port N] [--dry-run] [--yes] [--force] [--wipe-storage]` | Take Kevaso off a machine again. Shows what it will remove and keep, and asks first (`--yes`: don't ask). After that it stops `kevasod` and the fence and looks again before removing anything. Refused while silos are on it (`--force`: go ahead, their data is lost). `--wipe-storage`: also remove the silo storage on a real disk. Nothing under Kevaso's paths is removed while something is mounted there. [Reset a machine]({{ '/workers/#reset-a-machine' | relative_url }}) |

Every command that logs in takes `--ssh-key PATH` (a key with a passphrase is taken from `ssh-agent`
or asked for) or `--password` (typed once, read from stdin when there's no terminal; kvoadm then adds
a key of its own), `--ssh-port N`, and `--host-key SHA256:…` (without it, you're asked to confirm
the fingerprint).

`init` and `host add` also take `--user NAME` (the account to log in as, when not in `USER@`),
`--worker` on `init` (the control-plane host runs silos too), `--egress-extra-ca FILE` (a PEM file
the egress gate also trusts, for a company TLS proxy) and `--openshell-gateway-scope host|namespace|silo`.

`init` and `host add` also take the extra hardening, each only when asked for:
`--disable-password-login` (SSH keys only) and `--change-root-password` (a random one nobody
knows). In a terminal without these flags they ask; `--no-extra-hardening` skips the questions.
Everything that asks has a flag, so every command runs unattended too.

Both look at the machine first and stop, changing nothing, if something won't work there;
`--dry-run` stops after looking.

### Ingress

| Command | Does |
|---|---|
| `kvoadm ingress install [--class NAME] [--driver nginx\|caddy\|kong] [--public] [--host-port internal\|external \| --lb-class CLASS] [--image REF] [--runtime NAME] [--option K=V]…` | Start a stock proxy as a silo and the `IngressClass` that drives it. Defaults: class `public`, driver `nginx`, runtime `docker`, ports on the host's external IP. [HTTPS sites]({{ '/ingress/' | relative_url }}) |

### People

| Command | Does |
|---|---|
| `kvoadm users invite NAME [--namespace NS [--role editor\|admin\|viewer]] [--group G]… [--ttl D]` | A one-time sign-in code |
| `kvoadm users list` | Everyone who can sign in |
| `kvoadm users disable NAME` | Cut someone off now |

### Keys

| Command | Does |
|---|---|
| `kvoadm kms rotate [USER@]ADDR --ssh-key KEY` | A new master key for secrets |
| …`--new-recovery-key` | Also replace the recovery key (it leaked) |

### Backups

| Command | Does |
|---|---|
| `kvoadm backup [USER@]ADDR --ssh-key KEY --output FILE` | A backup of the control plane, sealed to the recovery key |
| `kvoadm init [USER@]ADDR … --restore FILE` | A fresh machine from a backup; asks for the recovery key (or reads it from stdin), then reconnects the workers. It starts a new history of the same cluster, which workers accept as a deliberate restore. Silos on workers that the restored state does not know are kept, never deleted. [Backup and restore]({{ '/operations/backup/' | relative_url }}) |
| `kvoadm plugin install [USER@]ADDR --name N --binary FILE [--credential NAME=FILE]… [--config FILE] [--side controller\|node] [--manifest FILE [--yes]]` | A provider plugin as a hardened service (credentials read from files, root-only on the host). A manifest's privileges are shown and need `--yes`. A folder the plugin asks to write is one of those, and only its own folders are allowed. [Plugins]({{ '/extending/plugins/#what-a-manifest-asks-for' | relative_url }}) |
| `kvoadm plugin install [USER@]ADDR --package DIR [--binary FILE] [--api-server URL] [--adopt] [--yes]` | A [plugin operator]({{ '/extending/plugins/#plugin-operators' | relative_url }}): shows every rule its controller gets, installs on `yes` |
| `kvoadm plugin uninstall NAME [--force]` | Remove an operator and what it owns; refused while objects of its kinds exist unless `--force` |
| `kvoadm plugin list` | Operators: version, kinds, whether the controller runs, host |
| `kvoadm plugin remove [USER@]ADDR --name N` | Removes it: unit, program, credentials |

---

## kevasod: on the hosts

You rarely run it yourself; `kvoadm` installs it as a service.

| Command | Does |
|---|---|
| `kevasod control-plane --name N [--internal-ip IP] [--external-ip IP] [--dns-suffix SUFFIX] [--worker --runtime R… [--storage-device /dev/DISK \| --loop-storage SIZE]]` | The control plane (and worker). Give at least one of the two IPs. Refuses to start on a machine with no cluster record, or whose store is gone: [Starting a cluster]({{ '/operations/backup/#starting-a-cluster-and-a-node-that-finds-its-data-gone' | relative_url }}) |
| `kevasod worker --name N --server URL[,URL…] [--storage-device /dev/DISK \| --loop-storage SIZE]` | A worker-only host. Several URLs: every system node; it fails over between them |
| `kevasod fence-watch [--fence-after D] [--dir DIR] [--runtime R]…` | A worker's [fence]({{ '/silos/control-plane-loss/#the-fence' | relative_url }}): kills the silos with `onControlPlaneLoss: Stop` when the agent's last contact is older than each one's `after`. `--fence-after` (default `30s`) has no effect on the records the agent writes. Its own unit, `kevaso-fence.service` |
| `kevasod version` | Its version |
| `kevasod platforms` | The operating systems this build supports (`kvoadm` reads the same list). [Supported operating systems]({{ '/setup/prepare/#supported-operating-systems' | relative_url }}) |
| `kevasod admin recovery-key` / `kms-rotate [--new-recovery-key]` / `backup` | Through the admin socket, as root |
| `kevasod admin create-cluster --cluster NAME [--data DIR]` | Make this machine belong to cluster NAME: writes its record and nothing else (`kvoadm init` runs it). Does nothing if the record already names NAME; refused if it names another cluster. `kevasod control-plane` never creates a cluster |
| `kevasod admin restore [--data DIR]` | Put a backup into a fresh data directory and mark it a deliberate restore (`kvoadm init --restore` runs it) |
| `kevasod control-plane --dns-suffix SUFFIX` | The last label of the cluster's names (default `kevaso`): `SERVICE.NAMESPACE.SUFFIX`. [Addresses, ports and names]({{ '/networking/model/' | relative_url }}) |
| `kevasod control-plane --store sqlite\|etcd` | Where objects live: SQLite (default) or an embedded etcd member. The first `etcd` start moves `store.db` into it. [Operations]({{ '/operations/#the-store' | relative_url }}) |
| `kevasod control-plane --etcd-client-listen ADDR` | With `--store etcd`: the member's client address (default `127.0.0.1:2379`; keep it on loopback) |
| `kevasod control-plane --etcd-peer-listen ADDR` / `--etcd-advertise-peer ADDR` | With `--store etcd`: where other members connect, and what they dial. Without the first, no peer port is opened |
| `kevasod control-plane --join URL[,URL…] [--join-token-file F] [--ca-file F]` | First start of a new system node: join the cluster through one of these. `kvoadm host add --install control-plane` writes this |
| `kevasod control-plane --state-only` | A light system node: the etcd member only. Needs `--store etcd` |
| `kevasod control-plane --cluster-dns NAME` | A name with one record per system node: in the certificate, and the one sign-in address |
| `kevasod control-plane --lease-max D` / `--lease-default-max D` | The longest a [leased port]({{ '/networking/leases/#what-a-lease-limits' | relative_url }}) may be open (default `24h`), and the longest for a silo that sets no `lease.max` (default `1h`) |
| `kevasod control-plane --approval-wait D` / `--approval-max-wait D` / `--approval-access D` / `--approval-max-access D` | How long a request for a yes waits (default `10m`, at most `1h`), and how long approved access lasts (default `15m`, at most `12h`) |
| `kevasod control-plane --plugins-dir DIR` | Where provider plugins' sockets are (default `/run/kevaso-plugins`) |
| `kevasod control-plane --cert-controller N=PROGRAM` / `--dns-provider N=PROGRAM` | Register programs (also picked up from `/etc/kevaso/cert-controllers/` and `/etc/kevaso/dns-providers/`) |
| `kevasod dev` | The API without authentication on loopback, for trying `kvo` locally |
