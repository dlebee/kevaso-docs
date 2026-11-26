---
title: Ingress controllers and certificates
permalink: /admin/ingress/
---

# Ingress controllers and certificates

An Ingress is written by the people who run the site, in their namespace: that is
[HTTPS sites]({{ '/use/ingress/' | relative_url }}). Before one can work, an administrator makes the
**controller** (a silo running a stock proxy), its **class** (which host names each namespace may
publish), and the **Issuers** certificates come from. This page does those.

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

Where certificates come from is each Ingress's choice, among the Issuers [below](#where-certificates-come-from).

> **Publishing on a public class is its own permission:** `create` on `ingresses/publish-public`
> in the namespace. Namespace `admin`s have it, `editor`s don't, so nobody puts something on the
> internet by accident. Give it with a role ([People and access]({{ '/admin/access/#roles' | relative_url }})), or apply
> those Ingresses yourself with `kvoadm kvo apply`.

---

## Where certificates come from

An **Issuer** names a **cert-controller**: the part that turns a request for some host names into a
certificate. Kevaso does the rest: it decides when a certificate is needed and when to renew it,
makes the private key, and keeps the result in a Secret. A cert-controller only ever gets a
certificate signing request (CSR), never the key.

Built in:

| Cert-controller | Gets certificates from |
|---|---|
| `acme` | An ACME server: Let's Encrypt, or your own. Proves each host with an HTTP-01 or DNS-01 solver |
| `internal-ca` | A CA of its own, for private names. No proof needed. See [The internal CA](#the-internal-ca) |

You can add your own on the control-plane host: see [Other tools](#other-tools-certbot-and-friends).

### The internal CA

`internal-ca` signs with its own CA, the files `internal-ca.crt` and `internal-ca.key` in
`/var/lib/kevaso/pki/`. It is not the cluster CA. A client that pins the cluster CA does not trust
it. The public certificate is served at `/internal-ca.crt`.

An API server's name is never signed: the host's name, its addresses, and the host of its API URL.

```text
host "203.0.113.10" is a name of this cluster's API server and is not signed
```

Any other name on a Certificate is kept only when an IngressClass assigns it to that namespace, or
it is one of the namespace's own service names (`NAME.payments.kevaso`, or with `.mesh`,
`.internal` or `.external` before the suffix). Otherwise:

```text
host "api.example.com" is not assigned to namespace payments on an ingress class, and is not one of its service names (NAME.payments.kevaso): ask a cluster admin to assign it
```

A worker reads `/internal-ca.crt`. A system node's own worker already has the certificate.

- A `docker` or `podman` silo gets it in the CA bundle the host mounts at
  `/etc/ssl/certs/kevaso-ca-bundle.crt`, together with the host's system CAs and the egress CA.
  `SSL_CERT_FILE`, `REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE` and `NODE_EXTRA_CA_CERTS` point at that file.
- An OpenShell gateway gets it in `/etc/kevaso/openshell-upstream-cas.pem`, which the gateway's own
  user can read, and trusts that file upstream. The file is public certificates only. It is not
  kept in `/var/lib/kevaso`, which is root-only.

A microVM does not get this bundle. Its trust store is the image's, plus the egress CA when a grant
injects a secret.

### Your own ACME server

An Issuer can point at any ACME server, with its CA if it's private:

```yaml
apiVersion: kevaso/v1
kind: Issuer
metadata: { name: company-acme }
spec:
  acme:
    server: https://acme.internal.example.com/directory
    email: ops@example.com
    serverCA: |
      -----BEGIN CERTIFICATE-----
      …
```

Use it from an Ingress with `tls: { issuer: company-acme }`.

### Solvers: proving you own a name

Before an ACME server signs, you prove you control each host name. An Issuer lists **solvers**;
each host uses the most specific one that matches it.

| Solver | How the proof works | Good for |
|---|---|---|
| `http01` | The ACME server fetches a token from `http://HOST/.well-known/acme-challenge/…`; the ingress controller answers | Any public host pointing at the controller. The default |
| `dns01` | You publish a `TXT` record at `_acme-challenge.HOST` | Wildcards (`*.example.com`), and hosts nothing can reach on port 80 |

```yaml
apiVersion: kevaso/v1
kind: Issuer
metadata: { name: letsencrypt-dns }
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    solvers:
      - http01: {}
      - selector:
          dnsZones: [internal.example.com]
        dns01:
          provider: bind
      - selector:
          dnsNames: [status.example.com]
        dns01:
          provider: bind
          propagationSeconds: 120
```

Which solver a host gets:

1. one whose `dnsNames` lists it exactly;
2. otherwise the one with the longest matching `dnsZones` (`db.internal.example.com` and
   `*.internal.example.com` are both in the zone `internal.example.com`);
3. otherwise one with no `selector`.

A host that matches none fails, and `kvo describe certificate` says so.

`propagationSeconds` is how long Kevaso waits after writing the record before asking the ACME
server to check it (default 60). Raise it if your DNS host is slow to publish.

### DNS providers

A dns01 solver names a **DNS provider**: a program on the control-plane host that writes and
removes `TXT` records at your DNS host. Kevaso runs it like this:

```
PROGRAM present _acme-challenge.status.example.com. VALUE
PROGRAM cleanup _acme-challenge.status.example.com. VALUE
```

The name ends with a dot. It's the same convention as lego's `exec` provider, so scripts written
for lego work here.

Any program in `/etc/kevaso/dns-providers/` is a provider, named after its file. Names are lowercase letters, digits and dashes. A provider program is stopped after 2 minutes. For example, for a
DNS server that accepts dynamic updates (BIND, Knot, PowerDNS), save this as
`/etc/kevaso/dns-providers/bind` on the control-plane host:

```sh
#!/bin/sh
set -eu
case "$1" in
  present) action=add ;;
  cleanup) action=delete ;;
  *) exit 2 ;;
esac
nsupdate -k /etc/kevaso/bind-update.key <<EOF
server ns1.example.com
update $action $2 60 TXT "$3"
send
EOF
```

Make it root's and not writable by anyone else, then restart Kevaso so it's picked up:

```bash
sudo chown root:root /etc/kevaso/dns-providers/bind
```

```bash
sudo chmod 0755 /etc/kevaso/dns-providers/bind
```

```bash
sudo systemctl restart kevasod
```

The control plane runs these programs as root, so it refuses to start if one of them, or the
folder, can be changed by another user.

> **Why programs on the host, and not in the Issuer?** Otherwise anything that can create an
> Issuer could make the control plane run a command. An Issuer can only *name* a provider that
> whoever runs the host installed. DNS API credentials stay in files on that host, next to the
> program, never in the API.

### Other tools: certbot and friends

When you'd rather use a tool you already have, register it as a cert-controller. Any program in
`/etc/kevaso/cert-controllers/` is one, named after its file. Kevaso runs it as `PROGRAM issue` (stopped after 10 minutes) and
writes the request to its standard input as JSON:

```json
{
  "issuer": "certbot",
  "config": { "email": "ops@example.com" },
  "hosts": ["api.example.com"],
  "csr": "-----BEGIN CERTIFICATE REQUEST-----\n…"
}
```

It reads the certificate chain back from the program's standard output, in PEM, the site's
certificate first. Whatever the program prints to standard error goes to Kevaso's log. `config` is
whatever the Issuer gives it.

For example, certbot with a DNS plugin for your DNS host (here `DNSHOST`), as `/etc/kevaso/cert-controllers/certbot`:

```sh
#!/bin/sh
set -eu
[ "$1" = issue ] || exit 2
work=$(mktemp -d)
trap 'rm -rf "$work"' EXIT

request=$(cat)
echo "$request" | jq -r .csr > "$work/request.pem"
email=$(echo "$request" | jq -r .config.email)

certbot certonly --non-interactive --agree-tos --email "$email" \
  --csr "$work/request.pem" \
  --dns-DNSHOST --dns-DNSHOST-credentials /etc/kevaso/dnshost.ini \
  --config-dir /var/lib/kevaso/certbot --work-dir "$work/run" --logs-dir /var/log/kevaso-certbot \
  --cert-path "$work/cert.pem" --chain-path "$work/chain.pem" \
  --fullchain-path "$work/fullchain.pem" >&2

cat "$work/fullchain.pem"
```

Same ownership rules as DNS providers. Then an Issuer that uses it:

```yaml
apiVersion: kevaso/v1
kind: Issuer
metadata: { name: certbot }
spec:
  controller: certbot
  config:
    email: ops@example.com
```

certbot gets only the CSR, so it never sees the key Kevaso keeps.
