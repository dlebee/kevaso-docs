---
title: Ingress controllers
permalink: /admin/ingress/
# Sections that moved: an old link with one of these anchors goes to the new place.
moved_anchors:
  where-certificates-come-from: /admin/certificates/#where-certificates-come-from
  the-internal-ca: /admin/certificates/#the-internal-ca
  your-own-acme-server: /admin/certificates/#your-own-acme-server
  solvers-proving-you-own-a-name: /admin/certificates/#solvers-proving-you-own-a-name
  dns-providers: /admin/certificates/#dns-providers
  other-tools-certbot-and-friends: /admin/certificates/#other-tools-certbot-and-friends
---

# Ingress controllers

An Ingress is written by the people who run the site, in their namespace: that is
[HTTPS sites]({{ '/use/ingress/' | relative_url }}). Before one can work, an administrator makes the
**controller** (a silo running a stock proxy) and its **class** (which host names each namespace may
publish). This page does those. The **Issuers** certificates come from are on
[Certificates]({{ '/admin/certificates/' | relative_url }}).

The pieces, and how a request reaches a silo, are on [HTTPS sites]({{ '/use/ingress/#the-pieces' | relative_url }}).

---

## 1 · The controller

One command makes a controller silo and its class:

```bash
kvoadm ingress install --public --host api.example.com=payments
```

```text
→ Namespace kevaso-system
→ Silo ingress-public
→ IngressClass public
Ingress class public: nginx (docker.io/library/nginx:1.27) in silo kevaso-system/ingress-public, on its host's external IP. Kevaso writes its config from outside;
see it with: kvo get ingressclass public -o yaml
```

A host name is published only when the class assigns it to the Ingress's namespace. Without
`--host`, the command still makes the class and then says no host is assigned yet. An Ingress is
refused until one is.

It runs as an admin of your current `kvo` context. Anything that already exists is left as it is.
The silo is in the namespace `kevaso-system`, named `ingress-CLASS`, with 1 CPU and 256Mi.

| Flag | Meaning | Default |
|---|---|---|
| `--class NAME` | The IngressClass to make | `public` |
| `--driver NAME` | The proxy: `nginx`, `caddy` or `kong` | `nginx` |
| `--public` | The class is reachable from the internet (publishing on it needs its own permission, below) | private |
| `--host-port internal\|external` | Open the proxy's ports at its worker, on the worker's internal or external address (a `HostPort` service) | `external` |
| `--provider PROVIDER` | Put a balancer from that installed provider in front instead (a `LoadBalancer` service; not with `--host-port`) | |
| `--image REF` | The proxy's image | The driver's vendor image |
| `--runtime NAME` | The silo's runtime | `docker` |
| `--option KEY=VALUE` | A driver option, repeatable (see below) | |
| `--host NAME=NAMESPACE[,NAMESPACE]` | Assign that host name to those namespaces. Repeat the flag for another name. `NAME` may be a one-label suffix, `*.example.com` | none: the class assigns no host, and an Ingress is refused until one is |
| `--cluster NAME` | The cluster folder to act from | The only one, or `$KVOADM_CLUSTER` |

The host that runs it needs the runtime and, with `--host-port external`, an external IP.

### The drivers

| Driver | Image | Ports | How a change takes effect |
|---|---|---|---|
| `nginx` | `docker.io/library/nginx:1.27` | 80, 443 | One `nginx.conf`, checked with `nginx -t`, then `nginx -s reload` |
| `caddy` | `docker.io/library/caddy:2.8` | 80, 443 | A Caddyfile with Caddy's automatic HTTPS off, checked with `caddy validate`, then `caddy reload` |
| `kong` | `docker.io/library/kong:3.7` | 8000, 8443 | Kong without a database: the whole config sent to its admin API, which checks and swaps it in one step |

`kvoadm ingress install` is made for `nginx` and `caddy`. Kong's image needs `KONG_DATABASE=off` and an
admin listener, so for Kong you bring your own silo and name it in the class (below). The admin API is
called from the host on the silo IP, so keep its port closed to everything else.

**Options** go in the class (`spec.controller.options`, or `--option`). A name the driver doesn't know,
or a value it won't take, is refused.

| Driver | Option | Meaning (default) |
|---|---|---|
| `nginx` | `user` | The user worker processes run as (`nginx`) |
| | `pid` | The pid file the image's nginx uses (`/run/nginx.pid`) |
| | `worker-connections` | Per worker (`1024`) |
| | `client-max-body-size` | Largest request body (`16m`) |
| | `proxy-connect-timeout` | To a backend (`5s`) |
| | `proxy-read-timeout` | Between two reads from a backend (`60s`) |
| | `proxy-send-timeout` | Between two writes to a backend (`60s`) |
| | `http2` | On 443, nginx 1.25.1 or later: `true` or `false` (`false`) |
| `caddy` | `dial-timeout` | To a backend (`5s`) |
| | `read-header-timeout` | For a client's request headers (`30s`) |
| | `idle-timeout` | A kept-alive client connection (`2m`) |
| `kong` | `admin-port` | Where Kong's admin API listens in the silo (`8001`) |
| | `connect-timeout`, `read-timeout`, `write-timeout` | To a backend (`5s`), between two reads (`60s`), between two writes (`60s`) |

**Annotations** go on an Ingress, prefixed with the driver's name, and only that driver reads them. An
annotation for another driver is ignored.

| Driver | Annotation | Meaning |
|---|---|---|
| `nginx` | `nginx/proxy-body-size` | Largest request body on this Ingress's routes |
| | `nginx/proxy-connect-timeout`, `nginx/proxy-read-timeout`, `nginx/proxy-send-timeout` | Timeouts for this Ingress |
| | `nginx/ssl-redirect` | `false`: also serve these routes on plain HTTP instead of redirecting |
| | `nginx/buffering` | `false`: stream responses as they come (server-sent events) |
| `caddy` | `caddy/dial-timeout` | To a backend |
| | `caddy/flush-interval` | `-1`: stream responses as they come, or a duration such as `500ms` |
| | `caddy/ssl-redirect` | `false`: also serve these routes on plain HTTP |
| `kong` | `kong/connect-timeout`, `kong/read-timeout`, `kong/write-timeout` | Timeouts for this Ingress |
| | `kong/strip-path` | `true`: drop the matched path prefix before passing the request on |
| | `kong/ssl-redirect` | `false`: also serve these routes on plain HTTP |

### Your own controller silo

Any silo can be the controller: run the proxy image as shipped, open its ports, and name the silo in
the class. `controller.siloSet` names a SiloSet instead: every instance gets the same configuration.

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: ingress-edge, namespace: kevaso-system }
spec:
  runtime: docker
  container: { image: docker.io/library/nginx:1.27 }
  resources: { cpu: 1, memory: 256Mi }
  ports:
    - { name: http,  port: 80 }
    - { name: https, port: 443 }
  services:
    - type: HostPort                 # worker: external
      ports: [{ port: http }, { port: https }]
```

One silo is the controller of one class.

## 2 · The class

```yaml
apiVersion: kevaso/v1
kind: IngressClass
metadata: { name: public }
spec:
  controller:
    driver: nginx
    namespace: kevaso-system
    silo: ingress-public
    options: { client-max-body-size: 32m }
  public: true
  hosts:
    - { host: api.example.com, namespaces: [payments] }
    - { host: "*.example.com", namespaces: [payments, shop] }
```

| Field | Meaning |
|---|---|
| `controller.driver` | The proxy's driver: `nginx`, `caddy` or `kong` |
| `controller.namespace` | The namespace of the controller silo (required) |
| `controller.silo` or `controller.siloSet` | The silo, or the SiloSet whose instances all run the proxy (one of the two) |
| `controller.options` | The driver's options |
| `public` | Reachable from the internet |
| `hosts` | Which namespaces may publish which names. `host` is a host name, or a one-label suffix such as `*.example.com` (that matches `a.example.com`, and not `example.com` or `b.a.example.com`). `namespaces` lists the namespaces allowed |
| `dns` | `{ provider, ttl }`: publish every host name of this class through a DNS provider, pointing at the controller's address. `ttl` is in seconds (300). The provider comes from a [provider plugin]({{ '/admin/extending/plugins/' | relative_url }}); its page says how |

Only a cluster admin writes an IngressClass, which is where the assignment lives. On a class that
is already there:

```bash
kvoadm ingress assign public --host api.example.com=payments
```

```text
→ IngressClass public assigns api.example.com to payments
```

`--host` is `NAME=NAMESPACE[,NAMESPACE]`, and it can be repeated. The same flag on
`kvoadm ingress install` records the assignment on the class it creates. A class that already
exists is left as it is; the command tells you to add the host with `ingress assign`.

Where certificates come from is each Ingress's choice, among the Issuers on [Certificates]({{ '/admin/certificates/' | relative_url }}).

> **Publishing on a public class is its own permission:** `create` on `ingresses/publish-public`
> in the namespace. Namespace `admin`s have it, `editor`s don't, so nobody puts something on the
> internet by accident. Give it with a role ([People and access]({{ '/admin/access/#roles' | relative_url }})), or apply
> those Ingresses yourself with `kvoadm kvo apply`.
