---
title: Sets, images and updates
permalink: /silos/sets/
---

# Sets, images and updates

Many identical silos are a **SiloSet**: a scale set of stateful machines, each a normal silo with a
stable name. They start from the block in their template (`container:` or `vm:`). A standalone **Service** gives them one name and one
address. Updates are deliberate: a silo is *reset* onto a new image, never silently replaced.

---

## What a silo starts from

A silo says what it starts from in a block named for what the image is. Exactly one block, and the
one its runtime takes:

| Runtime | Block | What it names |
|---|---|---|
| `docker`, `podman`, `openshell`, `openshell-container` | `container:` | A registry image |
| `cloud-hypervisor` | `vm:` | A bootable disk |

Admission refuses the wrong block and says which one to use.

### `container:`

`container.image` is a registry reference, used as it is. Nothing is created first.

```yaml
kind: Silo
spec:
  runtime: docker
  container:
    image: docker.io/library/python:3.12-slim
    command: [python3, -m, http.server, "8080"]      # optional: the main process
    pull: { secretRef: registry-pull }                # optional: a private registry's credential
```

| Field | Means |
|---|---|
| `image` | A registry reference, or the built-in `ubuntu-24.04` (which is `docker.io/library/ubuntu:24.04`). It can't be the name of a VMImage. On `openshell` and `openshell-container` a rootfs archive (`.tar`, `.tar.gz`, `.tgz`) is refused |
| `command` | The main process. Default: the image's own |
| `pull.secretRef` | A `registry` secret in the silo's namespace. Default: the namespace's default pull secret for the image's registry, if there is one. Docker and podman only: the OpenShell runtimes refuse it |

Pin the image by digest (`…@sha256:…`) if it must never change under you: a tag can move.
`kvo describe silo` flags an image that isn't pinned, and shows the digest that was actually pulled.
How a registry credential is made and kept is on [Secrets]({{ '/secrets/' | relative_url }}).

### `vm:`

A microVM boots from a disk. Give either `image` or `disk`, not both.

```yaml
kind: Silo
spec:
  runtime: cloud-hypervisor
  vm: { image: web-app:v2 }          # a VMImage's version, or web-app for its newest
---
kind: Silo
spec:
  runtime: cloud-hypervisor
  vm:
    disk:                            # a disk stated inline
      url: https://images.example.com/web-app-v2.img
      sha256: 4f0c…                  # 64 hex digits: every host checks what it downloads
```

| Field | Means |
|---|---|
| `image` | `NAME[:VERSION]` of a VMImage in the silo's namespace, or the built-in `ubuntu-24.04`. It can't be a registry reference |
| `disk.url` | An `https://` address. The worker downloads it. The body is capped at `disk.size`, or at the default disk (20 GiB, 21474836480 bytes) when `disk.size` is omitted. A larger body is refused and the partial file is removed |
| `disk.sha256` | The disk's hash. Hosts refuse a disk that doesn't match |
| the file itself | Raw, or a plain qcow2 with no backing file, no external data file and no encryption. Another format is refused. [Runtimes]({{ '/silos/runtimes/#cloud-hypervisor' | relative_url }}) |

A download that names its length over the cap fails before the body is kept:

```text
GET https://images.example.com/web-app.img: 21474836481 bytes, larger than the disk's 21474836480 bytes
```

A body that runs past the cap without that length fails as:

```text
GET https://images.example.com/web-app.img: body is larger than the disk's 21474836480 bytes
```

That number is the disk this silo will get: `disk.size`, or 21474836480 (20 GiB) when `disk.size`
is omitted. A body the same size as the disk is kept.

Every address, after DNS and including a redirect, is refused when it is a cluster address,
loopback, link-local or a private address. The message names the host it was about to use:

```text
GET 127.0.0.1: loopback
GET 10.64.0.5: the silo range
GET 10.62.0.1: the mesh range
GET 10.65.0.1: the service range
GET 192.168.1.1: a private address
GET 169.254.169.254: link-local
```

An address in `10.0.0.0/8` that is outside those cluster ranges says
`a private address: the mesh, silos and services are in it`. The silo's `Ready` condition is false,
reason `StartFailed`, and the message is `image URL: ` plus one of these, then
`(trying again in …)`.

### VMImage: a named disk with versions

A **VMImage** is a named list of disk versions, for microVM silos. It saves repeating a URL and
hash, and lets a set roll to a new release by changing one word.

```yaml
apiVersion: kevaso/v1
kind: VMImage
metadata: { name: web-app, namespace: ops }
spec:
  versions:                                    # oldest first
    - name: v1
      disk:
        url: https://images.example.com/web-app-v1.img
        sha256: 9a31…
    - name: v2
      disk:
        url: https://images.example.com/web-app-v2.img
        sha256: 4f0c…
```

```bash
kvo get vmimages -n ops
```

A version never changes once added, so a silo on `v1` always boots the same disk; a new release is
a new version. A VMImage is only ever a disk: container images live in registries, and Kevaso keeps
no second list of them. A silo that names a VMImage that doesn't exist, or a version it doesn't
have, stays `Pending` with the condition `Image` false, reason `ImageMissing`, and the message says what is missing.

`kvo describe silo NAME` shows what its source resolved to, and what it runs right now.

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

- failover (`onHostLoss` other than `Wait`). A move is by hand, with `kvo move … --confirm-fenced`,
  once you know the old copy is stopped;
- extra instances during a set's rollout (`maxSurge`, `StartFirst`). An exclusive instance is
  always updated one at a time, stop-first.

`exclusive` does not stop the silo when its worker loses the control plane: the silo keeps running
there, and Kevaso starts no other copy. To have it stop itself, add `onControlPlaneLoss: Stop` on a
cluster whose control plane is always reachable ([When the control plane is lost]({{ '/silos/control-plane-loss/#the-opt-in-stop-when-the-control-plane-is-lost' | relative_url }})).

> A single silo can't use `update.strategy: StartFirst` at all: Kevaso refuses it, since it would
> need the disk cloned. A set can use it, by starting an extra instance instead.

### Readiness

A silo can say when it is ready, so that a set's rollout waits for it and traffic waits too. That is
[probes and routing]({{ '/silos/probes/' | relative_url }}).

---

## SiloSets

```yaml
apiVersion: kevaso/v1
kind: SiloSet
metadata: { name: api-pool, namespace: payments }
spec:
  instances: 3
  spread: host                  # different hosts where possible (or zone, region)
  standby: 1                    # an extra one, booted and idle, ready to take over
  upgradePolicy: Rolling        # or Manual (the default)
  template:                     # each instance's silo spec
    labels: { app: api }
    runtime: docker
    container: { image: docker.io/library/python:3.12-slim }   # what every instance starts from
    volumes: [{ name: data, mountPath: /data }]
    ports: [{ name: http, port: 8080 }]
    service:
      ports: [{ port: http }]
```

The instances are `api-pool-0`, `api-pool-1` and `api-pool-2`, each its own silo, with its own disk
and volumes. Nothing is replaced to "heal" one: if it breaks, its host restarts it in place.

```bash
kvo get silosets -n payments
```

```bash
kvo describe siloset api-pool -n payments
```

### Growing and shrinking

```bash
kvo scale siloset api-pool --instances 5 -n payments
```

When it grows, a standby instance is used first, since it's booted already; then new ones are
made, and the standby is refilled.

Shrinking deletes instances and their disks, so `kvo scale` lists which ones first and wants
`--yes`. With no volumes in the template, instances that aren't Ready go first, then the newest
(`scaleInPolicy: NewestFirst`, the default, or `OldestFirst`). When the template has volumes, the
newest instances past the desired count go first, even when an older instance is not Ready. Those
are the extra ones from a surge (`maxSurge`). The scale-in policy still orders within that group.
A standby trim still takes instances that aren't Ready first. A protected instance is never removed:

```bash
kvo protect silo api-pool-0 -n payments
```

### Rolling out a new image

The set has no image of its own: what instances start from is the template's `container.image`,
`vm.image` or `vm.disk`. Change that (or anything else in the template) and apply it. What happens depends on `upgradePolicy`:

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
    - { name: http, port: 8080, targetPort: http }   # a port name the silos declare (default: this port's name)
```

- It gets a virtual IP (its cluster IP) and the name `api.payments.kevaso`.
- A port is sent only to a silo that declares it (`targetPort`, or the port's own name) with the same protocol. A LoadBalancer also holds that silo unless the silo opens the port in its own service, and not only on a lease. `kvo describe service` shows the reason.
- Every worker sends that address to one of its endpoints: the Ready silos its selector picks.
  Standby instances are never endpoints.
- The connection keeps the caller's address, so **each endpoint's `allowFrom` still decides**.
  A silo the instances don't allow is refused through the Service too.

```bash
kvo describe service api -n payments      # its address and endpoints
```

### An outside address

`type: LoadBalancer` with a provider's load-balancer class gives the Service an address from outside.

```yaml
spec:
  type: LoadBalancer
  loadBalancerClass: lb-class
```

- The address is an [IP]({{ '/networking/ips/' | relative_url }}): by default the namespace's shared one for the
  class. `loadBalancerIP: NAME` picks one you keep, and `loadBalancerDedicated: true` makes one of its own.
- The load balancer's backends are the hosts that have endpoints.
- Each host sends the traffic only to its own endpoints, so they see the load balancer's address,
  not the client's.
- The instances themselves stay private.
- It needs `create` on `services/expose-public` (namespace admins have it), including an edit that
  widens who can reach it ([People and access]({{ '/access/#roles' | relative_url }})).
- `routeWhen` says whether an endpoint gets traffic only while ready (`ready`) or as soon as it runs
  (`running`). It is required when the silos it selects have a readiness probe; an endpoint that is held
  shows in `kvo describe service`. See [probes and routing]({{ '/silos/probes/' | relative_url }}).
- Only classes that spread over hosts fit. A shared IP that follows one silo is for a silo's own
  service. See the [provider page]({{ '/providers/linode/' | relative_url }}).

Full examples are in the repo: `examples/07-siloset-service.yaml` and `examples/08-images-and-registries.yaml`.
