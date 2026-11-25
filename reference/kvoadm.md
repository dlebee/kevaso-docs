---
title: kvoadm and kevasod commands
permalink: /reference/kvoadm/
audience: admin
---

# kvoadm and kevasod commands

Every administrator's command on one page: `kvoadm`, which works on your machines over SSH and as
the cluster admin, and `kevasod`, the program it installs on them. `kvoadm` picks the cluster with
`--cluster NAME`, else `$KVOADM_CLUSTER`, else the only folder in `~/.kevaso/clusters/`.
`kvoadm kvo …` runs any [kvo command]({{ '/reference/kvo/' | relative_url }}) with the admin certificate.

---

## kvoadm: running the platform

### Machines

| Command | Does |
|---|---|
| `kvoadm init [USER@]ADDR --ssh-key KEY --host-key SHA256:… --kevasod BIN [--name N] [--cluster C] [--api-server URL] [--store sqlite\|etcd] [--worker] [--storage-device /dev/DISK \| --loop-storage SIZE] [--replace-unit]` | Create the cluster (the only command that does), install the control plane, write the cluster folder (`~/.kevaso/clusters/C/`, your admin certificate) and your kvo context (a plain user, `editor` in every namespace), print the recovery key. `--store etcd`: start on etcd, ready for [more system nodes]({{ '/admin/operations/ha/' | relative_url }}). `--worker`: the machine runs silos too. A second run keeps the installed worker, runtimes, storage, store, host name, addresses, etcd peer listen and join flags. A different value is refused unless `--replace-unit`. [Starting a cluster]({{ '/admin/operations/backup/#starting-a-cluster-and-a-node-that-finds-its-data-gone' | relative_url }}) |
| `kvoadm host enable-worker [USER@]ADDR --ssh-key KEY [--runtime R]… [--loop-storage SIZE \| --storage-device /dev/DISK]` | The control-plane host runs silos too |
| `kvoadm host reconnect HOST\|[USER@]ADDR [--ssh-key KEY] [--server URL]` | Point a worker at the control plane again (after a restore onto another address) |
| `kvoadm host trust-control-plane [USER@]ADDR --ssh-key PATH [--host-key SHA256:…] [--ssh-port N] [--data DIR] [--yes]` | Set aside a worker's record of its control plane and restart `kevasod`, so it trusts the control plane it reaches next. It does this for an unreadable record, or for a hold on an older control plane. A readable record that is not a hold is left alone. A hold on an older control plane ends only this way, or when `kvoadm init --restore` starts a new history. [An unreadable record]({{ '/admin/control-plane-loss/#an-unreadable-record' | relative_url }}) |
| `kvoadm host kept [USER@]ADDR --ssh-key PATH [--host-key SHA256:…] [--ssh-port N] [--data DIR] [--remove NAMESPACE/NAME/UID [--yes]]` | Silo copies this worker still has though the control plane doesn't know them (a delete it never saw, or a restore that forgot them). `--remove` deletes that copy, its disk and its volumes, after you type `NAMESPACE/NAME` (`--yes`: don't ask). At least the UID's first 8 characters |
| `kvoadm host add [USER@]ADDR --ssh-key KEY --kevasod BIN --install worker [--name N] [--region R] [--zone Z] [--label K=V] [--runtime R] [--behind-nat] [--internal-ip IP] [--external-ip IP] [--storage-device /dev/DISK \| --loop-storage SIZE]` | Add a worker. `--storage-device`: a whole, empty disk for the silos' volume group, when the host has none. `--loop-storage`: development only, a sparse file instead. With neither and no `kevaso` volume group, the microVM runtime stops with a message. [Where silo disks live]({{ '/admin/workers/#where-silo-disks-live' | relative_url }}) |
| `kvoadm host add [USER@]ADDR … --install control-plane [--state-only] [--cluster-dns NAME] [--internal-ip IP] [--external-ip IP]` | A second or third [system node]({{ '/admin/operations/ha/#add-a-system-node' | relative_url }}); `--state-only`: a light node (a vote only) |
| `kvoadm host install HOST control-plane [--state-only] [--ssh-key KEY]` | Make an existing worker a system node too |
| `kvoadm host uninstall HOST control-plane` | Take the control plane off a host that is a worker too (refused if it would lose the store's majority) |
| `kvoadm host remove HOST [--confirm-off]` | Remove a host from the cluster. With silos still on it, it lists which start elsewhere and which stay `HostDeleted`, then asks whether the machine is off or wiped (`--confirm-off`: yes). A host an open restore still needs cannot be removed without that confirmation. Its old certificate is refused afterwards. A system node: out of the store first; refused if it would lose the majority. [Removing a host]({{ '/admin/workers/host-loss/#removing-a-host' | relative_url }}) |
| `kvoadm host confirm-off NAME [--yes\|--confirm-off]` | Say a host the restored history is waiting on is off or wiped. Asks `Is NAME off or wiped? [y/N]` unless `--yes` or `--confirm-off`. The name does not have to be a Host object. [Backup and restore]({{ '/admin/operations/backup/#3--wait-until-each-silo-is-settled' | relative_url }}) |
| `kvoadm store migrate [USER@]ADDR --ssh-key KEY` | Move a SQLite control plane onto etcd with a peer listener |
| `kvoadm upgrade --kevasod BIN [--kevasod BIN…] [--ssh-key KEY] [--batch N] [--version V] [--dry-run]` | Roll a new `kevasod` over the cluster: system nodes (light first, leader last), then workers. One `--kevasod` per operating system and CPU in the cluster; a host with none refuses the roll before any change. A binary that does not report its own version, or reports a different one from `--version`, is refused. Build it with `-ldflags "-X kevaso.com/core/internal/version.Version=vX.Y.Z"`. [Rules]({{ '/admin/operations/ha/#upgrade' | relative_url }}) |
| `kvoadm host list` | Hosts, both of their IPs, and their state |
| `kvoadm host addresses HOST add\|remove CIDR…` | The extra addresses routed to a host, which the built-in `ip-pool` class hands out ([Addresses you keep]({{ '/admin/ips/' | relative_url }})). `remove` is refused for an address an IP holds |
| `kvoadm host addresses HOST list` | Each range, and which IP holds each address |
| `kvoadm host drain NAME` | Stop new silos landing on the host, and start a fresh copy elsewhere of each silo with `onHostLoss: Recreate`. The others are listed and stay. [Draining a host]({{ '/admin/workers/host-loss/' | relative_url }}) |
| `kvoadm host uncordon NAME` | The host takes new silos again |
| `kvoadm host ssh NAME [add ADDR… [--local] \| remove ADDR…]` | A host's SSH addresses, in the order `kvoadm` tries them. `add` puts them in the team's `cluster.yaml`, `--local` in your `local.yaml` (tried first). [A host's addresses]({{ '/admin/access/#a-hosts-addresses' | relative_url }}) |
| `kvoadm admins [list] \| add NAME KEY.pub… \| remove NAME \| sync [--dry-run]` | The admins in `cluster.yaml` and their public keys; `sync` puts them on every host's `authorized_keys` and takes removed ones off. [Admins]({{ '/admin/access/#admins' | relative_url }}) |
| `kvoadm kvo ARGS…` | Any `kvo` command, as the cluster's admin (the certificate in the cluster folder) |
| `kvoadm reset HOST\|[USER@]ADDR [--ssh-key KEY] [--host-key SHA256:…] [--ssh-port N] [--dry-run] [--yes] [--force] [--wipe-storage]` | Take Kevaso off a machine again. Shows what it will remove and keep, and asks first (`--yes`: don't ask). After that it stops `kevasod` and the fence and looks again before removing anything. Refused while silos are on it (`--force`: go ahead, their data is lost). Volumes on Kevaso's loop file with no silo on the host are refused unless `--wipe-storage`. `--wipe-storage` also removes the silo storage on a real disk. When Docker was already on the machine and resetting changes its daemon.json, Docker is restarted, and containers that were running and are not Kevaso's are started again. A failed step fails the reset and keeps the install record. Nothing under Kevaso's paths is removed while something is mounted there. [Reset a machine]({{ '/admin/workers/#reset-a-machine' | relative_url }}) |

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
| `kvoadm ingress install [--class NAME] [--driver nginx\|caddy\|kong] [--public] [--host-port internal\|external \| --lb-class CLASS] [--host NAME=NAMESPACE[,NAMESPACE]]… [--image REF] [--runtime NAME] [--option K=V]…` | Start a stock proxy as a silo and the `IngressClass` that drives it. `--host` assigns a name (or a one-label suffix `*.example.com`) to those namespaces. Defaults: class `public`, driver `nginx`, runtime `docker`, ports on the host's external IP. A host is published only when the class assigns it. [HTTPS sites]({{ '/use/ingress/' | relative_url }}) |
| `kvoadm ingress assign CLASS --host NAME=NAMESPACE[,NAMESPACE]…` | Add host assignments to an IngressClass that is already there. `NAME` may be a suffix `*.example.com` |

### People

| Command | Does |
|---|---|
| `kvoadm users invite NAME [--group G]… [-n NS --role role/NAME\|clusterrole/NAME \| --cluster-role NAME] [--ttl D] [--cluster NAME]` | A one-time sign-in code, and the `kvo login` line to send with it. Cluster-admin only. The role flags are those of `kvo invite`, which does the same from `kvo`; with none, no role. `--ttl` defaults to 15m, at most 168h. [Invite someone]({{ '/admin/access/#invite-someone' | relative_url }}) |
| `kvoadm users list` | Everyone who can sign in |
| `kvoadm users disable NAME` | Cut someone off now |
| `kvoadm users resign HOST\|[USER@]ADDRESS --name USER [--kvo]` | Sign a new client certificate over SSH: the admin's, into the cluster folder, or with `--kvo` the plain user's, into the kvo context. The key is made on this computer. Use it when `kvo` says the client certificate has expired, or is nearly expired and could not be renewed. [Contexts]({{ '/admin/access/#your-own-kvo-and-the-admin-certificate' | relative_url }}) |

### Keys

| Command | Does |
|---|---|
| `kvoadm kms rotate [USER@]ADDR --ssh-key KEY` | A new master key for secrets |
| …`--new-recovery-key` | Also replace the recovery key (it leaked) |

### Backups

| Command | Does |
|---|---|
| `kvoadm backup [USER@]ADDR --ssh-key KEY --output FILE` | A backup of the control plane, sealed to the recovery key |
| `kvoadm init [USER@]ADDR … --restore FILE (--no-hosts-added \| --host-added NAME)` | A fresh machine from a backup; asks for the recovery key (or reads it from stdin), then reconnects the workers. Exactly one of `--no-hosts-added` and repeatable `--host-added NAME`. It starts a new history of the same cluster. Workers may stop silos. They do not start a copy that is not already running until that silo is settled. Prints four lines: `restore: history ID`, `hosts that have not reported`, `unsettled silos`, `frozen silos`. Silos the restored state does not know are kept, never deleted. [Backup and restore]({{ '/admin/operations/backup/' | relative_url }}) |
| `kvoadm restore status` | The same four lines, after the restore |
| `kvoadm restore settle SILO -n NS --to HOST` | Place a frozen silo on a host that reported a copy. Cluster-admin only. Other copies stay stopped with their disks. The same write is `kvo restore settle SILO --to HOST [-n NS]` |
| `kvoadm plugin install [USER@]ADDR --name N --binary FILE [--cluster-secret NAME]… [--config FILE] [--side controller\|node] [--manifest FILE [--yes]]` | A provider plugin as a hardened service. Each `--cluster-secret` names a cluster secret. Install creates the service account of the side it installs, `plugin-NAME` for the controller or `plugin-NAME-node` for `--side node`, in namespace `kevaso-system` (only a cluster admin may change objects there), and prints the account to bind. A worker fetches only what the node side's account may read. A ClusterRole reads a value only with `get` on `clustersecrets/value` and `resourceNames` listing that name; bind that role to the account. The host fetches each cluster secret's keys and keeps them root-only, a folder per name, and the plugin reads each key as `NAME_KEY` from its credential directory. Repeat the flag for each name. A manifest's privileges are shown and need `--yes`. A folder the plugin asks to write is one of those, and only its own folders are allowed. [Plugins]({{ '/admin/extending/plugins/#install-a-provider-plugin' | relative_url }}) |
| `kvoadm plugin install [USER@]ADDR --package DIR [--binary FILE] [--api-server URL] [--adopt] [--yes]` | A [plugin operator]({{ '/admin/extending/plugins/#plugin-operators' | relative_url }}): shows every rule its controller gets, installs on `yes` |
| `kvoadm plugin uninstall NAME [--force]` | Remove an operator and what it owns; refused while objects of its kinds exist unless `--force` |
| `kvoadm plugin list` | Operators: version, kinds, whether the controller runs, host |
| `kvoadm plugin remove [USER@]ADDR --name N [--side controller\|node]` | Removes one side (default `controller`): its unit, and the program, cached secrets and service account once no side is left |

---

## kevasod: on the hosts

You rarely run it yourself; `kvoadm` installs it as a service.

| Command | Does |
|---|---|
| `kevasod control-plane --name N [--internal-ip IP] [--external-ip IP] [--dns-suffix SUFFIX] [--worker --runtime R… [--storage-device /dev/DISK \| --loop-storage SIZE]]` | The control plane (and worker). Give at least one of the two IPs. Refuses to start on a machine with no cluster record, or whose store is gone: [Starting a cluster]({{ '/admin/operations/backup/#starting-a-cluster-and-a-node-that-finds-its-data-gone' | relative_url }}) |
| `kevasod worker --name N --server URL[,URL…] [--storage-device /dev/DISK \| --loop-storage SIZE]` | A worker-only host. Several URLs: every system node; it fails over between them |
| `kevasod fence-watch [--fence-after D] [--dir DIR] [--runtime R]…` | A worker's [fence]({{ '/admin/control-plane-loss/#the-fence' | relative_url }}): kills the silos with `onControlPlaneLoss: Stop` when the agent's last contact is older than each one's `after`. `--fence-after` (default `30s`) has no effect on the records the agent writes. Its own unit, `kevaso-fence.service` |
| `kevasod version` | Its version |
| `kevasod platforms` | The operating systems this build supports (`kvoadm` reads the same list). [Supported operating systems]({{ '/admin/setup/prepare/#supported-operating-systems' | relative_url }}) |
| `kevasod admin recovery-key` / `kms-rotate [--new-recovery-key]` / `backup` | Through the admin socket, as root |
| `kevasod admin create-cluster --cluster NAME [--data DIR]` | Make this machine belong to cluster NAME: writes its record and nothing else (`kvoadm init` runs it). Does nothing if the record already names NAME; refused if it names another cluster. `kevasod control-plane` never creates a cluster |
| `kevasod admin restore [--data DIR]` | Put a backup into a fresh data directory and mark it a deliberate restore (`kvoadm init --restore` runs it). With no host statement it still opens a new history; no silo from before that history settles until a statement exists |
| `kevasod admin sign-user --name NAME [--role user]` | On a control-plane host, as root: sign a user's certificate request from stdin (a cluster-admin's, or with `--role user` a plain user's: only what RBAC binds to the name). Prints JSON `{certificate, ca}`. `kvoadm users resign` runs this over SSH and writes the certificate into the context. [Contexts]({{ '/admin/access/#your-own-kvo-and-the-admin-certificate' | relative_url }}) |
| `kevasod control-plane --dns-suffix SUFFIX` | The last label of the cluster's names (default `kevaso`): `SERVICE.NAMESPACE.SUFFIX`. [Addresses, ports and names]({{ '/use/networking/model/' | relative_url }}) |
| `kevasod control-plane --store sqlite\|etcd` | Where objects live: SQLite (default) or an embedded etcd member. The first `etcd` start moves `store.db` into it. [Operations]({{ '/admin/operations/#the-store' | relative_url }}) |
| `kevasod control-plane --etcd-client-listen ADDR` | With `--store etcd`: the member's client address (default `127.0.0.1:2379`; keep it on loopback) |
| `kevasod control-plane --etcd-peer-listen ADDR` / `--etcd-advertise-peer ADDR` | With `--store etcd`: the member listens on its internal address. It also listens on the reach address when that address is on one of its own interfaces, and it advertises the reach address. Under 1:1 NAT it listens on the internal address only. Without `--etcd-peer-listen`, no peer port is opened |
| `kevasod control-plane --join URL[,URL…] [--join-token-file F] [--ca-file F]` | First start of a new system node: join the cluster through one of these. `kvoadm host add --install control-plane` writes this |
| `kevasod control-plane --state-only` | A light system node: the etcd member only. Needs `--store etcd` |
| `kevasod control-plane --cluster-dns NAME` | A name with one record per system node: in the certificate, and the token issuer. Without it the issuer is this node's reach address (external IP, or internal IP). A restore onto a new address then invalidates tokens. [Backup and restore]({{ '/admin/operations/backup/#1--a-new-machine' | relative_url }}) |
| `kevasod control-plane --lease-max D` / `--lease-default-max D` | The longest a [leased port]({{ '/use/networking/leases/#what-a-lease-limits' | relative_url }}) may be open (default `24h`), and the longest for a silo that sets no `lease.max` (default `1h`) |
| `kevasod control-plane --approval-wait D` / `--approval-max-wait D` / `--approval-access D` / `--approval-max-access D` | How long a request for a yes waits (default `10m`, at most `1h`), and how long approved access lasts (default `15m`, at most `12h`) |
| `kevasod control-plane --plugins-dir DIR` | Where provider plugins' sockets are (default `/run/kevaso-plugins`) |
| `kevasod control-plane --cert-controller N=PROGRAM` / `--dns-provider N=PROGRAM` | Register programs (also picked up from `/etc/kevaso/cert-controllers/` and `/etc/kevaso/dns-providers/`) |
| `kevasod dev` | The API without authentication on loopback, for trying `kvo` locally |
