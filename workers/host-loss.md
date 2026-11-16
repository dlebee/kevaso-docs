---
title: Draining a host
permalink: /workers/host-loss/
---

# Draining a host

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
