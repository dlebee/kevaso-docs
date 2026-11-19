---
title: Draining and removing a host
permalink: /workers/host-loss/
---

# Draining and removing a host

Machines fail and need maintenance. This page is about taking one out on purpose.

## When a host goes away

A host that stops reporting for **40 seconds** is marked `Unknown`. Nothing is moved because of it:
its silos keep running if it is only cut off, and Kevaso starts no other copy until you say the old
one is off, or a silo has opted in to be stopped and restarted elsewhere. What you see, what to do
and the opt-in are on one page:
[When the control plane is lost]({{ '/silos/control-plane-loss/' | relative_url }}).

---

## Maintenance: draining

```bash
bin/kvoadm host drain hv2
```

`drain` takes the host's name (the `--context NAME` flag, last, picks another `kvo` context). It
has no other flags. It does two things:

1. **Cordons** the host: it takes no new silos. `kvoadm host list` shows it as `cordoned`.
2. **Hands off** each silo on it that has `onHostLoss: Recreate`, the way `kvo move` does.
   The silo loses its host and is placed again by the scheduler, with `movingFrom: hv2` set. The new
   host starts it only after `hv2`'s agent has stopped it and let go. So two copies never run.

Silos with `Wait` or `Move` stay where they are. `drain` names the policy:
`NAMESPACE/NAME stays: onHostLoss Move pins it to hv2 (its disk is local)`. Delete them, change their
policy, or move them yourself with `kvo move silo NAME --to HOST`
([Moves]({{ '/silos/moves/' | relative_url }})). A container (`docker`, `podman`,
`openshell-container`) is not moved that way: a plain `kvo move` is refused. It needs
`onHostLoss: Recreate` and `kvo move … --recreate`. When none stay, it prints that the
host is empty of silos once the moved ones are gone.

`drain` does not wait. It returns once it has written the new state. Watch the silo with
`kvo describe silo NAME`: it shows `Moving from: hv2 (it stops the silo and lets go first)`,
and the `Ready` condition says `Drained` and then `Moving` (`waiting for hv2 to stop it and let go`).
When `hv2` has let go, `Moving from` disappears and the silo starts on its new host.

A silo that `drain` hands off starts from its image: its own disk is not carried over. That is what
`Recreate` means.

Bring the host back with:

```bash
bin/kvoadm host uncordon hv2
```

### When the old host is dead

The new host waits for the old host's agent to stop the silo and let go. If `hv2` is down, nobody
answers for it, so the silo stays in `Moving`, and Kevaso does not assume the old copy is off.
Once you know it is, end the hand-off with:

```bash
kvo move silo NAME -n NAMESPACE --to HOST --confirm-fenced
```

A move that is already under way is refused without `--confirm-fenced` when its old host is lost
(`a move is already under way (from hv2, which is lost): its host is lost: move it only once the old
copy is provably stopped …`). `kvoadm host drain` has no `--confirm-fenced` flag: drain a host while
it is up. [Moving a silo off a host you know is off]({{ '/silos/control-plane-loss/#moving-a-silo-off-a-host-you-know-is-off' | relative_url }}).

### What has run

`drain` has run on two bare-metal machines. Ending a hand-off whose old host is lost has not run
on real machines.

## Removing a host

Removing a host takes it out of the cluster for good. A host with no silos is removed at once:

```bash
bin/kvoadm host remove hv2
```

A host that still has silos placed on it, or moving from it, is removed only once you confirm that
the machine is **off or wiped**. Kevaso can't stop those copies itself, so your confirmation is what
lets their silos start somewhere else, the same as `kvo move … --confirm-fenced`.
Deleting the Host a silo is placed on confirms that host only. A handoff whose source is lost
still needs `kvo move … --confirm-fenced`.

`kvoadm host remove` lists what will happen to each silo, then asks:

```text
→ Is hv2 off or wiped? [y/N]
```

To answer yes from a script, add `--confirm-off`:

```bash
bin/kvoadm host remove hv2 --confirm-off
```

`kvo delete host` asks for the same confirmation. Without `--confirm-off`, it is refused and names
how many silos are still there:

```bash
kvo delete host hv2 --confirm-off
```

While a restore is still waiting on a host, removing it is refused until you confirm it off the
same way. The account and `kvoadm host confirm-off` are on
[Backup and restore]({{ '/operations/backup/#3--wait-until-each-silo-is-settled' | relative_url }}).

### What happens to its silos

| The silo | After the host is removed |
|---|---|
| `onHostLoss: Recreate` | A fresh copy starts on another host, from its image |
| `onHostLoss: Move`, and every disk and volume can attach elsewhere | It starts on another host with its data |
| Anything else, for example a microVM with a local root disk | It stays in the cluster, not running anywhere, with the condition `Unschedulable`, reason `HostDeleted` |
| Moving from the removed host | The hand-off ends, and the silo starts on its new host |

Removing a host never deletes a volume. A local disk or volume on the removed machine is simply out
of reach, and the `HostDeleted` message says so. `Unschedulable`/`HostDeleted` is set only when a
fresh read says the Host is gone. A silo marked `HostDeleted` stays that way, even if
a machine joins again under the same name. The host that runs a silo cannot change `status.service`,
or the `Unschedulable`/`HostDeleted` condition. The control plane writes them.

To get it running again, start a fresh copy on another host:

```bash
kvo move silo NAME -n NAMESPACE --to HOST --recreate
```

Or delete it with `kvo delete silo NAME -n NAMESPACE`.

### The removed machine

The removed host's certificate is no longer accepted, from the moment its removal is recorded: its
open connections to the cluster are closed, and new ones are refused with
`host hv2 was removed from the cluster; join it again with kvoadm host add`.

If the machine is still on, its `kevasod` takes that refusal as final. It stops every silo it runs
(a normal stop, then a kill if that doesn't finish), keeps every disk and volume, and does nothing
more until it joins again. So a "yes, it's off" that was wrong ends with the old copies stopped,
not running next to the new ones.

To use the machine again, add it again. `kvoadm host add` moves the old certificate folder aside
(`/var/lib/kevaso/pki` gets a dated name; nothing is deleted) and joins with a new one:

```bash
bin/kvoadm host add root@198.51.100.20 \
    --ssh-key ~/.ssh/kevaso_ed25519 \
    --host-key SHA256:… \
    --kevasod bin/kevasod-linux-amd64 \
    --install worker \
    --name hv2
```

To start it clean instead, with the old silos' disks gone, reset it first:

```bash
bin/kvoadm reset root@198.51.100.20 \
    --ssh-key ~/.ssh/kevaso_ed25519 \
    --host-key SHA256:…
```

The new certificate it gets works. [Adding workers]({{ '/workers/' | relative_url }}#add-it). Removing a host is for cluster admins only.

System nodes keep their own rules: removing one is refused if the store would lose its majority
([High availability]({{ '/operations/ha/' | relative_url }})).

### What has run

Removing a stopped worker that still had a recreatable silo and a silo with a local volume has run
on two nested-VM machines. The first silo started on the other worker, the second was marked
`HostDeleted`, and the machine's old certificate was refused when its `kevasod` started again.
Then it was added again.
