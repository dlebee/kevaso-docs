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
| `kvo delete KIND NAME` | Delete it. A namespace is refused while it still holds anything, and while a binding names the group `system:serviceaccounts:NS`: delete those first. Deleting a namespace never deletes what is in it. `DELETE` on a `/status` path is refused (405) |
| `kvo delete host NAME --confirm-off` | Delete a Host that still has silos, confirming the machine is off or wiped. Refused without the flag. [Removing a host]({{ '/workers/host-loss/#removing-a-host' | relative_url }}) |
| `kvo watch KIND [-n NS \| -A]` | Print the current state, then every change |

Kinds: `namespaces` (`ns`), `hosts`, `silos`, `silosets` (`ss`), `services` (`svc`), `vmimages`
(`vmi`), `secrets`, `grants`, `openshellpolicies`, `ips`, `ingresses` (`ing`), `ingressclasses`, `certificates`
(`cert`), `issuers`, `roles`, `rolebindings`, `clusterroles`, `clusterrolebindings`, `clustersecrets`,
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
| `kvo move silo NAME --to HOST [--recreate] [--confirm-fenced [--insist]] [-n NS]` | Move it, with its data when its storage allows. A container's own files are never kept: a plain move is refused, and a fresh copy needs `onHostLoss: Recreate` and `--recreate`. `--recreate` is only for `onHostLoss: Recreate` silos. A plain move needs `create` on `silos/move`. `--confirm-fenced`: its host is lost and you know the old copy is off; without it a move from a lost host is refused. Refused while that host claimed a start of this silo less than 30 seconds ago; `--insist`, with `--confirm-fenced`, overrides that. Both flags need `create` on `hosts/confirm-off` (cluster-scoped; only `cluster-admin` by default). [Moves]({{ '/silos/moves/' | relative_url }}), [Moving a silo off a host you know is off]({{ '/silos/control-plane-loss/#moving-a-silo-off-a-host-you-know-is-off' | relative_url }}) |
| `kvo get silos [-o wide]` | `NAME`, `RUNTIME`, `PHASE`, `READY`, `HOST`, `AGE`. `-o wide` adds `MOVABLE` (where each silo could go) and `ON-CP-LOSS` (`stops after 2m` for a silo with `onControlPlaneLoss: Stop` and no `after` of its own, or `stops after` that `after`; `-` for the others). A silo on a silent host has phase `Unknown` |
| `kvo describe silo NAME` | Its state, [probes]({{ '/silos/probes/' | relative_url }}), what it starts from, what it does when its host loses the control plane (`On control-plane loss`), its conditions (`HostUnreachable`, `FenceArmed`, `FenceStopped`), and each port with its addresses and lease. [Addresses, ports and names]({{ '/networking/model/' | relative_url }}) |
| `kvo open silo NAME --port PORT [--port PORT]… [--for D] [--no-wait]` | Open [leased ports]({{ '/networking/leases/' | relative_url }}) for a while. Each port is its own lease, on its own terms. If a lease needs a yes, wait for it (`--no-wait`: don't). A refused port doesn't stop the others, and the command exits non-zero |
| `kvo close silo NAME --port PORT [--port PORT]…` | Close them early |
| `kvo get leases [-n NS \| -A]` | The leased ports and their state: `SILO`, `PORT`, `PHASE`, `WHEN`, `BY` |
| `kvo restore status` | The same four lines as `kvoadm restore status`. [Backup and restore]({{ '/operations/backup/#3--wait-until-each-silo-is-settled' | relative_url }}) |
| `kvo restore settle SILO --to HOST [-n NS]` | The same write as `kvoadm restore settle`. Cluster-admin only |

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
| `kvo create secret NAME --from-file F\|-` | Store a value. `--from-literal VALUE` is refused unless you add `--i-know`: the value is on the command line |
| `kvo create secret NAME --generate [--type ed25519\|ecdsa-p256]` | Let Kevaso make it |
| …`--allow MODES`, `--write-only`, `-l KEY=VALUE` | Restrict delivery (`broker`, `inject`, `file`, `env`, comma-separated), make write-only, label |
| `kvo create secret NAME --type registry --server HOST --username U --from-file - [--default-pull]` | A [private registry's]({{ '/secrets/' | relative_url }}) credential: write-only, the password on stdin. `--default-pull`: the one used for that registry when a silo names none |
| `kvo secret set NAME --from-file F\|-` | A new value (rotation) |
| `kvo get secret NAME --reveal` | Print the value (needs `read-value`; never for write-only) |
| `kvo create clustersecret NAME [--description TEXT] --from-file F\|-` | A secret with no namespace, for a plugin. Cluster-admin only. The value comes from a file or from stdin. `--from-literal` is refused |
| `kvo set clustersecret NAME --from-file F\|-` | Replace that value. Each host that has it writes the new copy and restarts the plugin |
| `kvo get clustersecret [NAME] [--reveal]` | The name and description. `--reveal` prints the value. A value is readable by a cluster admin, or by `get` on `clustersecrets/value` in a ClusterRole whose `resourceNames` lists that name |
| `kvo delete clustersecret NAME` | Delete it |
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
| `kvo login SERVER --ca sha256:…` | Sign in (browser + invite code). `SERVER` is `https://` only. Several control planes from an invite are comma-separated: each is tried for the CA, and the whole list is kept in the context |
| `kvo logout` | Forget this context's session |
| `kvo ctx [NAME]` | List contexts, or switch |
| `kvo ns [NAME]` | Show or set the default namespace |
| `kvo auth whoami` | Who you are to the cluster |
| `kvo auth can-i VERB RESOURCE[/NAME] [-n NS]` | Exits 0 for yes, 1 for no. `ssh`, `files`, `open`, `close` and `move` are silo subresources (`silos/ssh` is the terminal, not a silo named `ssh`) |
| `kvo create token SA [--duration 1h]` | A service-account token, for CI (`$KVO_TOKEN`). Default one hour, at most 2160h (90 days). A namespace editor can't mint one |

### Environment

| Variable | Means |
|---|---|
| `KVO_CONFIG` | The config file (default `~/.kevaso/config.yaml`). `kvoadm` doesn't read it |
| `KVO_TOKEN`, `KVO_CA_FILE` | A service-account token and the cluster CA, for CI |
| `KVO_SERVER` | A plain-HTTP server (`kevasod dev`, local experiments) |

---

## kvoadm: running the platform

### Machines

| Command | Does |
|---|---|
| `kvoadm init [USER@]ADDR --ssh-key KEY --host-key SHA256:… --kevasod BIN [--name N] [--cluster C] [--api-server URL] [--store sqlite\|etcd] [--worker] [--storage-device /dev/DISK \| --loop-storage SIZE] [--replace-unit]` | Create the cluster (the only command that does), install the control plane, write the cluster folder (`~/.kevaso/clusters/C/`, your admin certificate) and your kvo context (a plain user, `editor` in every namespace), print the recovery key. `--store etcd`: start on etcd, ready for [more system nodes]({{ '/operations/ha/' | relative_url }}). `--worker`: the machine runs silos too. A second run keeps the installed worker, runtimes, storage, store, host name, addresses, etcd peer listen and join flags. A different value is refused unless `--replace-unit`. [Starting a cluster]({{ '/operations/backup/#starting-a-cluster-and-a-node-that-finds-its-data-gone' | relative_url }}) |
| `kvoadm host enable-worker [USER@]ADDR --ssh-key KEY [--runtime R]… [--loop-storage SIZE \| --storage-device /dev/DISK]` | The control-plane host runs silos too |
| `kvoadm host reconnect HOST\|[USER@]ADDR [--ssh-key KEY] [--server URL]` | Point a worker at the control plane again (after a restore onto another address) |
| `kvoadm host trust-control-plane [USER@]ADDR --ssh-key PATH [--host-key SHA256:…] [--ssh-port N] [--data DIR] [--yes]` | Set aside a worker's record of its control plane and restart `kevasod`, so it trusts the control plane it reaches next. It does this for an unreadable record, or for a hold on an older control plane. A readable record that is not a hold is left alone. A hold on an older control plane ends only this way, or when `kvoadm init --restore` starts a new history. [An unreadable record]({{ '/silos/control-plane-loss/#an-unreadable-record' | relative_url }}) |
| `kvoadm host kept [USER@]ADDR --ssh-key PATH [--host-key SHA256:…] [--ssh-port N] [--data DIR] [--remove NAMESPACE/NAME/UID [--yes]]` | Silo copies this worker still has though the control plane doesn't know them (a delete it never saw, or a restore that forgot them). `--remove` deletes that copy, its disk and its volumes, after you type `NAMESPACE/NAME` (`--yes`: don't ask). At least the UID's first 8 characters |
| `kvoadm host add [USER@]ADDR --ssh-key KEY --kevasod BIN --install worker [--name N] [--region R] [--zone Z] [--label K=V] [--runtime R] [--behind-nat] [--internal-ip IP] [--external-ip IP] [--storage-device /dev/DISK \| --loop-storage SIZE]` | Add a worker. `--storage-device`: a whole, empty disk for the silos' volume group, when the host has none. `--loop-storage`: development only, a sparse file instead. With neither and no `kevaso` volume group, the microVM runtime stops with a message. [Where silo disks live]({{ '/workers/#where-silo-disks-live' | relative_url }}) |
| `kvoadm host add [USER@]ADDR … --install control-plane [--state-only] [--cluster-dns NAME] [--internal-ip IP] [--external-ip IP]` | A second or third [system node]({{ '/operations/ha/#add-a-system-node' | relative_url }}); `--state-only`: a light node (a vote only) |
| `kvoadm host install HOST control-plane [--state-only] [--ssh-key KEY]` | Make an existing worker a system node too |
| `kvoadm host uninstall HOST control-plane` | Take the control plane off a host that is a worker too (refused if it would lose the store's majority) |
| `kvoadm host remove HOST [--confirm-off]` | Remove a host from the cluster. With silos still on it, it lists which start elsewhere and which stay `HostDeleted`, then asks whether the machine is off or wiped (`--confirm-off`: yes). A host an open restore still needs cannot be removed without that confirmation. Its old certificate is refused afterwards. A system node: out of the store first; refused if it would lose the majority. [Removing a host]({{ '/workers/host-loss/#removing-a-host' | relative_url }}) |
| `kvoadm host confirm-off NAME [--yes\|--confirm-off]` | Say a host the restored history is waiting on is off or wiped. Asks `Is NAME off or wiped? [y/N]` unless `--yes` or `--confirm-off`. The name does not have to be a Host object. [Backup and restore]({{ '/operations/backup/#3--wait-until-each-silo-is-settled' | relative_url }}) |
| `kvoadm store migrate [USER@]ADDR --ssh-key KEY` | Move a SQLite control plane onto etcd with a peer listener |
| `kvoadm upgrade --kevasod BIN [--kevasod BIN…] [--ssh-key KEY] [--batch N] [--version V] [--dry-run]` | Roll a new `kevasod` over the cluster: system nodes (light first, leader last), then workers. One `--kevasod` per operating system and CPU in the cluster; a host with none refuses the roll before any change. A binary that does not report its own version, or reports a different one from `--version`, is refused. Build it with `-ldflags "-X kevaso.com/core/internal/version.Version=vX.Y.Z"`. [Rules]({{ '/operations/ha/#upgrade' | relative_url }}) |
| `kvoadm host list` | Hosts, both of their IPs, and their state |
| `kvoadm host addresses HOST add\|remove CIDR…` | The extra addresses routed to a host, which the built-in `ip-pool` class hands out ([Addresses you keep]({{ '/networking/ips/' | relative_url }})). `remove` is refused for an address an IP holds |
| `kvoadm host addresses HOST list` | Each range, and which IP holds each address |
| `kvoadm host drain NAME` | Stop new silos landing on the host, and start a fresh copy elsewhere of each silo with `onHostLoss: Recreate`. The others are listed and stay. [Draining a host]({{ '/workers/host-loss/' | relative_url }}) |
| `kvoadm host uncordon NAME` | The host takes new silos again |
| `kvoadm host ssh NAME [add ADDR… [--local] \| remove ADDR…]` | A host's SSH addresses, in the order `kvoadm` tries them. `add` puts them in the team's `cluster.yaml`, `--local` in your `local.yaml` (tried first). [A host's addresses]({{ '/access/#a-hosts-addresses' | relative_url }}) |
| `kvoadm admins [list] \| add NAME KEY.pub… \| remove NAME \| sync [--dry-run]` | The admins in `cluster.yaml` and their public keys; `sync` puts them on every host's `authorized_keys` and takes removed ones off. [Admins]({{ '/access/#admins' | relative_url }}) |
| `kvoadm kvo ARGS…` | Any `kvo` command, as the cluster's admin (the certificate in the cluster folder) |
| `kvoadm reset HOST\|[USER@]ADDR [--ssh-key KEY] [--host-key SHA256:…] [--ssh-port N] [--dry-run] [--yes] [--force] [--wipe-storage]` | Take Kevaso off a machine again. Shows what it will remove and keep, and asks first (`--yes`: don't ask). After that it stops `kevasod` and the fence and looks again before removing anything. Refused while silos are on it (`--force`: go ahead, their data is lost). Volumes on Kevaso's loop file with no silo on the host are refused unless `--wipe-storage`. `--wipe-storage` also removes the silo storage on a real disk. When Docker was already on the machine and resetting changes its daemon.json, Docker is restarted, and containers that were running and are not Kevaso's are started again. A failed step fails the reset and keeps the install record. Nothing under Kevaso's paths is removed while something is mounted there. [Reset a machine]({{ '/workers/#reset-a-machine' | relative_url }}) |

Every command except `init` acts on one cluster: `--cluster NAME`, else `$KVOADM_CLUSTER`, else the
only folder in `~/.kevaso/clusters/` (`$KVOADM_HOME` replaces `~/.kevaso`). Where a command takes
`[USER@]ADDR`, a host's name works too: `kvoadm` tries that host's addresses in order.

Every command that logs in takes `--ssh-key PATH` (a key with a passphrase is taken from `ssh-agent`
or asked for) or `--password` (typed once, read from stdin when there's no terminal; kvoadm then adds
a key of its own), `--ssh-port N`, and `--host-key SHA256:…` (without it, you're asked to confirm
the fingerprint).

`init` takes `--user NAME` (your name in Kevaso; default your login name on this computer) and
`--worker` (the control-plane host runs silos too). `init` and `host add` also take
`--egress-extra-ca FILE` (a PEM file the egress gate also trusts, for a company TLS proxy) and
`--openshell-gateway-scope host|namespace|silo`. `init --worker`, `host enable-worker` and `host add`
write those into the worker they install.

`init` and `host add` also take the extra hardening, each only when asked for:
`--disable-password-login` (SSH keys only) and `--change-root-password` (a random one nobody
knows). In a terminal without these flags they ask; `--no-extra-hardening` skips the questions.
Everything that asks has a flag, so every command runs unattended too.

Both look at the machine first and stop, changing nothing, if something won't work there;
`--dry-run` stops after looking.

### Ingress

| Command | Does |
|---|---|
| `kvoadm ingress install [--class NAME] [--driver nginx\|caddy\|kong] [--public] [--host-port internal\|external \| --lb-class CLASS] [--host NAME=NAMESPACE[,NAMESPACE]]… [--image REF] [--runtime NAME] [--option K=V]…` | Start a stock proxy as a silo and the `IngressClass` that drives it. `--host` assigns a name (or a one-label suffix `*.example.com`) to those namespaces. Defaults: class `public`, driver `nginx`, runtime `docker`, ports on the host's external IP. A host is published only when the class assigns it. [HTTPS sites]({{ '/ingress/' | relative_url }}) |
| `kvoadm ingress assign CLASS --host NAME=NAMESPACE[,NAMESPACE]…` | Add host assignments to an IngressClass that is already there. `NAME` may be a suffix `*.example.com` |

### People

| Command | Does |
|---|---|
| `kvoadm users invite NAME [--namespace NS [--role editor\|admin\|viewer]] [--group G]… [--ttl D]` | A one-time sign-in code. Cluster-admin only. With `--namespace` and no `--role`, the role is `editor`. `--ttl` defaults to 15m |
| `kvoadm users list` | Everyone who can sign in |
| `kvoadm users disable NAME` | Cut someone off now |
| `kvoadm users resign HOST\|[USER@]ADDRESS --name USER [--kvo]` | Sign a new client certificate over SSH: the admin's, into the cluster folder, or with `--kvo` the plain user's, into the kvo context. The key is made on this computer. Use it when `kvo` says the client certificate has expired, or is nearly expired and could not be renewed. [Contexts]({{ '/access/#contexts' | relative_url }}) |

### Keys

| Command | Does |
|---|---|
| `kvoadm kms rotate [USER@]ADDR --ssh-key KEY` | A new master key for secrets |
| …`--new-recovery-key` | Also replace the recovery key (it leaked) |

### Backups

| Command | Does |
|---|---|
| `kvoadm backup [USER@]ADDR --ssh-key KEY --output FILE` | A backup of the control plane, sealed to the recovery key |
| `kvoadm init [USER@]ADDR … --restore FILE (--no-hosts-added \| --host-added NAME)` | A fresh machine from a backup; asks for the recovery key (or reads it from stdin), then reconnects the workers. Exactly one of `--no-hosts-added` and repeatable `--host-added NAME`. It starts a new history of the same cluster. Workers may stop silos. They do not start a copy that is not already running until that silo is settled. Prints four lines: `restore: history ID`, `hosts that have not reported`, `unsettled silos`, `frozen silos`. Silos the restored state does not know are kept, never deleted. [Backup and restore]({{ '/operations/backup/' | relative_url }}) |
| `kvoadm restore status` | The same four lines, after the restore |
| `kvoadm restore settle SILO -n NS --to HOST` | Place a frozen silo on a host that reported a copy. Cluster-admin only. Other copies stay stopped with their disks. The same write is `kvo restore settle SILO --to HOST [-n NS]` |
| `kvoadm plugin install [USER@]ADDR --name N --binary FILE [--cluster-secret NAME]… [--config FILE] [--side controller\|node] [--manifest FILE [--yes]]` | A provider plugin as a hardened service. Each `--cluster-secret` names a cluster secret. Install creates the service account of the side it installs, `plugin-NAME` for the controller or `plugin-NAME-node` for `--side node`, in namespace `kevaso-system` (only a cluster admin may change objects there), and prints the account to bind. A worker fetches only what the node side's account may read. A ClusterRole reads a value only with `get` on `clustersecrets/value` and `resourceNames` listing that name; bind that role to the account. The host fetches each value and keeps it root-only, and the plugin reads that name from its credential directory. Repeat the flag for each name. A manifest's privileges are shown and need `--yes`. A folder the plugin asks to write is one of those, and only its own folders are allowed. [Plugins]({{ '/extending/plugins/#install-a-provider-plugin' | relative_url }}) |
| `kvoadm plugin install [USER@]ADDR --package DIR [--binary FILE] [--api-server URL] [--adopt] [--yes]` | A [plugin operator]({{ '/extending/plugins/#plugin-operators' | relative_url }}): shows every rule its controller gets, installs on `yes` |
| `kvoadm plugin uninstall NAME [--force]` | Remove an operator and what it owns; refused while objects of its kinds exist unless `--force` |
| `kvoadm plugin list` | Operators: version, kinds, whether the controller runs, host |
| `kvoadm plugin remove [USER@]ADDR --name N [--side controller\|node]` | Removes one side (default `controller`): its unit, and the program, cached secrets and service account once no side is left |

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
| `kevasod admin restore [--data DIR]` | Put a backup into a fresh data directory and mark it a deliberate restore (`kvoadm init --restore` runs it). With no host statement it still opens a new history; no silo from before that history settles until a statement exists |
| `kevasod admin sign-user --name NAME [--role user]` | On a control-plane host, as root: sign a user's certificate request from stdin (a cluster-admin's, or with `--role user` a plain user's: only what RBAC binds to the name). Prints JSON `{certificate, ca}`. `kvoadm users resign` runs this over SSH and writes the certificate into the context. [Contexts]({{ '/access/#contexts' | relative_url }}) |
| `kevasod control-plane --dns-suffix SUFFIX` | The last label of the cluster's names (default `kevaso`): `SERVICE.NAMESPACE.SUFFIX`. [Addresses, ports and names]({{ '/networking/model/' | relative_url }}) |
| `kevasod control-plane --store sqlite\|etcd` | Where objects live: SQLite (default) or an embedded etcd member. The first `etcd` start moves `store.db` into it. [Operations]({{ '/operations/#the-store' | relative_url }}) |
| `kevasod control-plane --etcd-client-listen ADDR` | With `--store etcd`: the member's client address (default `127.0.0.1:2379`; keep it on loopback) |
| `kevasod control-plane --etcd-peer-listen ADDR` / `--etcd-advertise-peer ADDR` | With `--store etcd`: the member listens on its internal address. It also listens on the reach address when that address is on one of its own interfaces, and it advertises the reach address. Under 1:1 NAT it listens on the internal address only. Without `--etcd-peer-listen`, no peer port is opened |
| `kevasod control-plane --join URL[,URL…] [--join-token-file F] [--ca-file F]` | First start of a new system node: join the cluster through one of these. `kvoadm host add --install control-plane` writes this |
| `kevasod control-plane --state-only` | A light system node: the etcd member only. Needs `--store etcd` |
| `kevasod control-plane --cluster-dns NAME` | A name with one record per system node: in the certificate, and the token issuer. Without it the issuer is this node's reach address (external IP, or internal IP). A restore onto a new address then invalidates tokens. [Backup and restore]({{ '/operations/backup/#1--a-new-machine' | relative_url }}) |
| `kevasod control-plane --lease-max D` / `--lease-default-max D` | The longest a [leased port]({{ '/networking/leases/#what-a-lease-limits' | relative_url }}) may be open (default `24h`), and the longest for a silo that sets no `lease.max` (default `1h`) |
| `kevasod control-plane --approval-wait D` / `--approval-max-wait D` / `--approval-access D` / `--approval-max-access D` | How long a request for a yes waits (default `10m`, at most `1h`), and how long approved access lasts (default `15m`, at most `12h`) |
| `kevasod control-plane --plugins-dir DIR` | Where provider plugins' sockets are (default `/run/kevaso-plugins`) |
| `kevasod control-plane --cert-controller N=PROGRAM` / `--dns-provider N=PROGRAM` | Register programs (also picked up from `/etc/kevaso/cert-controllers/` and `/etc/kevaso/dns-providers/`) |
| `kevasod dev` | The API without authentication on loopback, for trying `kvo` locally |
