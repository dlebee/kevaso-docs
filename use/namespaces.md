---
title: Namespaces
permalink: /use/namespaces/
---

# Namespaces

A **namespace** holds a team's things: silos and sets, services, ingresses, IPs and IP sets,
secrets, grants, roles. Names are unique inside one, and roles are given in one. An administrator
makes namespaces; you work in the ones you're invited to.

```bash
kvo ns payments
```

`kvo ns` makes `payments` your default, so later commands can leave out `-n`.

---

## Empty one: `kvo delete all`

`kvo delete all` deletes everything in a namespace, secrets included. The namespace itself stays,
ready to use again. It lists what is there and asks once:

```bash
kvo delete all -n payments
```

```text
namespace payments holds resources: 3 silos (2 with disks), 1 service, 2 kept IPs, 4 secrets.
This deletes all of them (stop silos, remove load balancers, release IPs, delete the rest, delete secrets and access); the namespace stays. Are you sure? [y/N]
```

- `--yes` is that yes, for a script. With no terminal to ask on and no `--yes`, it refuses and says
  so.
- It needs the right to delete each thing in it. It asks first, for every object, and if you may
  not delete one it names them and deletes nothing.
- Deleting silos deletes their disks and volumes. Kept IPs are released: their addresses go back.

---

## Delete one: `kvo delete namespace`

Deleting a namespace deletes what it holds, then the namespace. It shows what is left and asks once:

```bash
kvo delete namespace payments
```

```text
namespace payments still holds resources: 3 silos (2 with disks), 1 service, 2 kept IPs, 4 secrets.
Deleting the namespace deletes all of them first (stop silos, remove load balancers, release IPs, delete the rest, delete secrets and access). Are you sure? [y/N]
```

After the yes, `kvo` follows each step until the namespace is gone:

```text
step 1/5, stop silos: waiting on 3 silos
step 3/5, release IPs: waiting on 2 kept IPs
namespace/payments deleted
```

An empty namespace is deleted without a question. `--yes` skips it otherwise. Deleting a namespace
is a cluster admin's (`kvoadm kvo delete namespace …`).

### In this order

| Step | What goes |
|---|---|
| 1. Stop silos | SiloSets and silos, with their disks and volumes |
| 2. Remove load balancers | Services, Ingresses and leases: their balancers are deleted with them |
| 3. Release IPs | IP sets and IPs: provider addresses go back to the provider |
| 4. Delete the rest | Every other kind in it, built in or your own |
| 5. Delete secrets and access | Grants, secrets, role bindings, roles, service accounts |

Each step waits until what it deleted is gone before the next starts, so nothing is cut off from a
secret or an address while it still runs.

### If a step stops

The namespace stays `Terminating`, holding what is left, and the error names the step and why:

```text
namespace payments: deleting what is in it stopped at step 3/5, release IPs: … It stays Terminating; kvo delete namespace payments runs it again from there
```

Fix the cause (a provider that refused, a plugin that is down), then run the same command again:
it carries on from that step.

```bash
kvo describe namespace payments
```

shows where it is.

- The yes is for what was shown: if something new appears in the namespace before the delete starts,
  it is refused, and you are asked again.
- Through the API, a namespace that holds anything is deleted only with an explicit
  `?cascade=true`. A plain `DELETE` of a namespace that holds something is refused, as it always
  was.
