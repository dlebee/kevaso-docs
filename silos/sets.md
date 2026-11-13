---
title: Sets, images and updates
permalink: /silos/sets/
---

# Sets, images and updates

Many identical silos are a **SiloSet**: a scale set of stateful machines, each a normal silo with a
stable name. They start from an **Image**. A standalone **Service** gives them one name and one
address. Updates are deliberate: a silo is *reset* onto a new image, never silently replaced.

---

## Images

What a silo starts from depends on what kind of silo it is.

**A container silo** (docker, podman, OpenShell's container runtime) names its image the way you
would anywhere else: a registry reference. There is nothing to create first.

```yaml
kind: Silo
spec:
  runtime: docker
  image: docker.io/library/python:3.12-slim
```

Pin it by digest (`…@sha256:…`) if it must never change under you: a tag can move.
`kvo describe silo` flags an image that isn't pinned, and shows the digest that was actually pulled.
For a private registry, see [private registries]({{ '/secrets/' | relative_url }}).

**A microVM silo** (cloud-hypervisor) boots from a disk image. It names either the built-in one,
`ubuntu-24.04`, or an **Image** you define: a named list of disk versions.

```yaml
apiVersion: kevaso/v1
kind: Image
metadata: { name: web-app, namespace: ops }
spec:
  versions:                                    # oldest first
    - name: v1
      disk:
        url: https://images.example.com/web-app-v1.img
        sha256: 9a31…                          # every host checks what it downloads
    - name: v2
      disk:
        url: https://images.example.com/web-app-v2.img
        sha256: 4f0c…
```

```yaml
kind: Silo
spec:
  runtime: cloud-hypervisor
  image: web-app:v2          # or web-app, for the newest version
```

An Image's versions never change once added, so a silo on `v1` always boots the same disk; a new
release is a new version. An Image is only ever a disk: container images live in registries, and
Kevaso keeps no second list of them.

`kvo describe silo NAME` shows what its image resolved to, and what it runs right now.

---

## Updating a silo: a reset

Changing a silo's image, or running `kvo reset`, puts it back onto its image:

```bash
kvo reset silo api
```

1. The silo is stopped completely first (**stop-first**: never two copies).
2. Its disk (or container) is made fresh from the image.
3. Its name, address and **volumes are kept**. Its `firstBoot` init steps run again.

Anything you want to keep across resets belongs on a volume.

### `exclusive`: one copy, ever

If two copies of a silo must never run at the same time (a single-writer database, or a system with
equivocation penalties), set `exclusive: true`:

```yaml
spec:
  exclusive: true              # at most one copy may ever run
  availability: { onHostLoss: Wait }
```

Kevaso refuses:

- automatic failover (`onHostLoss` other than `Wait`);
- start-first updates (`update.strategy: StartFirst`);
- extra instances during a set's rollout.

> A single silo's `StartFirst` (start the new copy, switch, then stop the old one) needs its disk
> cloned, which comes with moves. Sets can already do it, by starting an extra instance instead.

### Readiness

"Running" isn't always "ready": a silo that is still starting up shouldn't get traffic. A readiness check is a
command run inside the silo now and then. Until it passes, the silo isn't Ready: no Service sends to
it, and a set's rollout waits for it.

```yaml
spec:
  readiness:
    command: [sh, -c, "curl -fs localhost:8080/health"]
    period: 15s                  # default 10s; each run times out after 5s
```

---

## SiloSets

```yaml
apiVersion: kevaso/v1
kind: SiloSet
metadata: { name: api-pool, namespace: payments }
spec:
  image: docker.io/library/python:3.12-slim
  instances: 3
  spread: host                  # different hosts where possible (or zone, region)
  standby: 1                    # an extra one, booted and idle, ready to take over
  upgradePolicy: Rolling        # or Manual (the default)
  template:                     # each instance's silo spec
    labels: { app: api }
    runtime: docker
    volumes: [{ name: data, mountPath: /data }]
    service:
      ports: [{ name: http, port: 8080 }]
```

The instances are `api-pool-0`, `api-pool-1` and `api-pool-2`, each its own silo, with its own disk
and volumes. Nothing is replaced to "heal" one: if it breaks, its host restarts it in place.

```bash
kvo get silosets -n payments
kvo describe siloset api-pool -n payments
```

### Growing and shrinking

```bash
kvo scale siloset api-pool --instances 5 -n payments
```

When it grows, a standby instance is used first, since it's booted already; then new ones are
made, and the standby is refilled.

Shrinking deletes instances and their disks, so `kvo scale` lists which ones first and wants
`--yes`. Instances that aren't Ready go first, then the newest (`scaleInPolicy: NewestFirst`, or
`OldestFirst`). A protected instance is never removed:

```bash
kvo protect silo api-pool-0 -n payments
```

### Rolling out a new image

Change the set's `image` (or its template) and apply it. What happens depends on `upgradePolicy`:

| Policy | What happens |
|---|---|
| **Manual** (default) | New instances get the new image. An existing one gets it when you reset it: `kvo reset silo api-pool-1`. |
| **Rolling** | Instances are reset onto it, a few at a time. |

How many at a time is `update`:

```yaml
spec:
  update:
    strategy: StartFirst     # start an extra instance on the new image first
    maxSurge: 1              # how many extra (StartFirst's default: 1)
    maxUnavailable: 0        # how many fewer than `instances` may be Ready (default 1)
```

With the default `StopFirst`, at most one instance is down at a time. With `StartFirst`, the set
never has fewer Ready instances than it asks for; the extra ones go again when the rollout is done.
Instances that are `exclusive` always go one at a time, stop-first.

To reset every instance onto the current image, whatever the policy:

```bash
kvo reset siloset api-pool --rolling -n payments
```

---

## Services

A standalone Service is one name and one address in front of many silos:

```yaml
apiVersion: kevaso/v1
kind: Service
metadata: { name: api, namespace: payments }
spec:
  selector: { app: api }
  ports:
    - { name: http, port: 8080, targetPort: http }   # a port name of the silos' own service, or a number
```

- It gets a virtual IP and the name `api.payments.kevaso`.
- Every worker sends that address to one of its endpoints: the Ready silos its selector picks.
  Standby instances are never endpoints.
- The connection keeps the caller's address, so **each endpoint's `allowFrom` still decides**.
  A silo the instances don't allow is refused through the Service too.

```bash
kvo describe service api -n payments      # its address and endpoints
```

### An outside address

`type: LoadBalancer` with a provider's load-balancer class gives the Service a public address.

```yaml
spec:
  type: LoadBalancer
  loadBalancerClass: linode-nodebalancer
```

- The load balancer's backends are the hosts that have endpoints.
- Each host sends the traffic only to its own endpoints, so they see the load balancer's address,
  not the client's.
- The instances themselves stay private.
- Like exposing a silo, it needs `create` on `services/expose-public` (namespace admins have it).
- Only classes that spread over hosts fit. A shared IP follows one silo, so it's for a silo's own
  service. See [Linode](../../providers/linode/).

A full example is in the repo: `examples/07-siloset-service.yaml`.
