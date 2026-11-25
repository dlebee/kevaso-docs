---
title: Probes and routing
permalink: /silos/probes/
---

# Probes and routing

> **None of this is required.** A silo with no probes is `Started` and `Ready` as soon as it runs,
> and Kevaso ensures nothing else about what runs inside it. Add probes when you want Kevaso to know
> when something has come up, can take work, or has gone bad.

A **probe** is a check the silo's host runs now and then. A silo can have up to three, each answering
one question:

| Probe | The question | While it hasn't passed |
|---|---|---|
| `startup` | Has it come up? | `readiness` and `liveness` don't run |
| `readiness` | Can it take work right now? | The silo isn't `Ready`: its service gets no traffic, and a set's rollout waits |
| `liveness` | Is it still healthy? | When it fails, the silo is `Degraded`. With `onFailure: restart` it is also restarted |

`routeWhen` on the silo's service says whether traffic waits for `readiness`.

---

## One probe

Each probe is exactly one of three checks, plus how often it runs:

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: api, namespace: payments }
spec:
  runtime: docker
  container: { image: "ghcr.io/acme/api:1.4" }
  ports: [{ name: web, port: 8080 }]
  probes:
    startup:   { tcp: { port: web }, period: 5s, timeout: 2m }
    readiness: { http: { port: web, path: /ready }, period: 10s }
    liveness:  { exec: { command: [/usr/local/bin/api, healthcheck] }, onFailure: restart }
  service:
    ports: [{ port: web, allowFrom: [client] }]
    routeWhen: ready
```

| Check | Passes when |
|---|---|
| `exec: { command: [ARGV…] }` | The command, run inside the silo, exits 0. It runs as given, with no shell: use `sh -c` for a script |
| `tcp: { port: NAME }` | The port accepts a connection |
| `http: { port: NAME, path: /P }` | A `GET` answers 2xx or 3xx. `path` defaults to `/` and must start with `/` |

`port` is the **name** of a port in the silo's `spec.ports`, not a number. Admission refuses a name
the silo doesn't declare.

| Field | Means | Default |
|---|---|---|
| `period` | How often it runs | `10s` |
| `timeout` | The limit for each run. On `startup` it is the **overall deadline** instead | `5s`; startup `5m` |
| `successes` | Results in a row that make it pass | `1` |
| `failures` | Results in a row that make it fail. Not on `startup` | `3` |
| `onFailure` | `liveness` only: `report` or `restart` | `report` |

`period` and `timeout` are durations like `10s` or `2m`. `successes` and `failures` can't be negative.
Because results in a row decide, one bad run doesn't flip the answer.

### Where a probe runs from

On the silo's own host, never from the control plane. `exec` goes through the runtime, as `kvo exec`
does. `tcp` and `http` connect from the host to the silo's own IP. That traffic needs no `allowFrom`
entry, no grant and no port opened on the host, and it opens nothing to anyone else. A probe says
whether the silo answers, not whether the outside can reach it.

A probe that can't run at all is `CantRun`, with its reason, and is **never counted as a failure**:

- the runtime can't run commands in the silo (`exec`);
- the silo has no address its host can reach: the OpenShell runtimes have no network, so only `exec`
  probes work on them.

A silo whose readiness probe can't run stays not `Ready` (reason `ReadinessCantRun`).

---

## Startup

`startup` gates the other two: until it passes, they are `Disabled`. It runs every `period`.

- If `timeout` (default 5 minutes) passes without it passing, the silo gets the condition `StartFailed`.
  The silo **stays up**, so you can look inside.
- The probe keeps running. If it passes late, the silo is `Started` after all.

## Readiness

Until `readiness` passes, the silo is not `Ready` (reason `NotReady`, with the last result). It
doesn't stop the silo and doesn't restart it: it only holds back what depends on being ready.

## Liveness

`liveness` runs once started. When it fails `failures` times in a row, the silo gets the condition
`Degraded`, which clears when the probe passes again.

`onFailure: restart` also restarts the silo, and only that setting does:

- The silo is stopped, then started again on the same disk and with the same address. Its init steps run
  as `init.when` says.
- Restarts back off: 10 seconds before the first, doubling up to 5 minutes. The wait starts over once the
  silo has stayed up for 10 minutes.
- `kvo describe silo` shows the count of restarts and the last one.

---

## `routeWhen`

Traffic to a silo's service is **stated, never assumed**. `spec.service.routeWhen` is one of:

| Value | Traffic goes to the silo |
|---|---|
| `ready` | Only while its readiness probe passes |
| `running` | As soon as it runs, whatever its probes say |

Rules admission applies:

- A silo with a `readiness` probe **and** a service must say `routeWhen`.
- `routeWhen: ready` needs a `readiness` probe.
- A silo with no `readiness` probe is `running`, and need not say it.
- A standalone Service has the same field, `spec.routeWhen`. It is refused when it selects, now, a silo or a
  SiloSet whose template breaks those rules. A silo that arrives later is held, with that reason.

### What "held" means on each path

While a silo is held, new traffic goes elsewhere or nowhere:

| Path | Held means |
|---|---|
| Its name | The name exists, with no address (an answer with no data) |
| A standalone Service | The silo is listed as held, not as an endpoint. No host sends to it, and it is no backend of the Service's load balancer |
| Its ports opened on its host (`hostPort`) and a load balancer's node ports | No rule on the host. Nothing already connected is cut |
| An Ingress route to it | Left out of the proxy's configuration |
| Another silo that its service lets in (`allowFrom`) | **Not held.** That is access, not routing |

Held never cuts a connection that is already open. Only a [lease]({{ '/networking/leases/' | relative_url }})
that ends does that. The two compose: a leased port is open only while its lease is open **and** the
silo is routed.

---

## What you see

```bash
kvo get silos -n payments
```

The `READY` column shows whether the silo is `Ready`.

```bash
kvo describe silo api -n payments
```

```text
Route when:  ready
Traffic:
  PORT      TRAFFIC
  web 8080  held since 40s (not ready: readiness hasn't passed yet: HTTP 503)
Probes:
  PROBE      CHECK                                  EVERY                         STATE                 LAST             REASON
  startup    tcp web                                5s, within 2m0s               Passing for 1m        Success 1m ago   -
  readiness  http web/ready                         10s                          Pending for 40s       Failure 4s ago   HTTP 503
  liveness   exec /usr/local/bin/api healthcheck    10s , on failure restart     Passing for 30s       Success 8s ago   -
Liveness restarts:  2, last 14m ago
```

A probe's state is `Pending` (not passed yet), `Passing`, `Failing`, `CantRun` or `Disabled` (waiting
for startup). `kvo describe service NAME` shows `Route when` and, for each endpoint, whether it is
`routed` or `held` and why.

The conditions on a silo, with probes: `Started`, `StartFailed` (only with a `startup` probe), `Ready`
and, from liveness or [upkeep]({{ '/silos/daemons/' | relative_url }}), `Degraded`.
