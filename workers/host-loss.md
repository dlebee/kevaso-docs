---
title: Host loss and draining
permalink: /workers/host-loss/
---

# Host loss and draining

Machines fail and need maintenance. What happens to their silos is yours to decide, per silo.

## When a host goes away

A host that stops reporting for **40 seconds** is marked lost. What happens to its silos is
**yours to choose**, per silo:

```yaml
spec:
  availability:
    onHostLoss: Wait        # Wait (default) | Recreate | Move (with its data: see Host loss and moves)
```

| Policy | Meaning | Good for |
|---|---|---|
| `Wait` | Stays where it is, down, until the host is back. Its disk is never lost. | Anything with state you care about |
| `Recreate` | Started fresh from its image on another host, 30 s after the host is marked lost. Its disk on the lost host is given up. | Stateless or rebuildable silos |
| `Move` | Started on another host with its data, 30 s after the host is marked lost. Every disk and volume must be on storage that can attach elsewhere. | Silos with data on movable storage |

To make sure two copies never run at once, a worker that loses the control plane for 30 seconds
**stops its own `Recreate` and `Move` silos** before the control plane would recreate them elsewhere,
even if `kevasod` itself has stopped: the small `kevaso-fence` service does it ([the fence]({{ '/silos/moves/#the-fence' | relative_url }})).

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

Silos with `Wait` or `Move` stay where they are: `drain` prints
`NAMESPACE/NAME stays: onHostLoss Wait pins it to hv2 (its disk is local)`. Delete them, change their
policy, or move them yourself with `kvo move silo NAME --to HOST`
([Host loss and moves]({{ '/silos/moves/' | relative_url }})). When none stay, it prints that the
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

The new host waits for the old host's agent. If `hv2` is down, nobody answers for it, so the silo
stays in `Moving` until `hv2` comes back and lets go. Nothing else clears `movingFrom`.

`kvo move` cannot get around this: it refuses a silo that has `movingFrom` set (`a move is already
under way`), and its `--confirm-fenced` flag, which is for a move from a lost host, never gets
looked at. `kvoadm host drain` has no `--confirm-fenced` flag. There is no command that
tells Kevaso the old host is stopped for a silo that is already handing off.

So drain a host while it is up. For a host that is already lost, a `Recreate` silo is placed
elsewhere by itself 30 s after the host is marked lost (above), without a handoff, and `kvo move
--confirm-fenced` is the way for a person to move a silo from a lost host
([the fence]({{ '/silos/moves/#the-fence' | relative_url }})).

### What has run

`drain` has been tested in-process, with stand-ins for the machines. It has not been run on real
machines.
