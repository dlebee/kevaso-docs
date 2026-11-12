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
    onHostLoss: Wait        # Wait (default) | Recreate
```

| Policy | Meaning | Good for |
|---|---|---|
| `Wait` | Stays where it is, down, until the host is back. Its disk is never lost. | Anything with state you care about |
| `Recreate` | Started fresh from its image on another host after 30 s. Its disk on the lost host is given up. | Stateless or rebuildable silos |

To make sure two copies never run at once, a worker that loses the control plane for 30 seconds
**stops its own `Recreate` silos** before the control plane would recreate them elsewhere.

---

## Maintenance: draining

```bash
bin/kvoadm host drain hv2
```

The host takes no new silos, and it shows which of its silos would move and which must wait (`Wait`
silos keep their disk there). Bring it back with:

```bash
bin/kvoadm host uncordon hv2
```
