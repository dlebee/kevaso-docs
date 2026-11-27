---
title: Certificates
permalink: /admin/certificates/
---

# Certificates

An Ingress gets its certificate from an **Issuer**, which an administrator sets up. This page is the
Issuers and the cert-controllers behind them. The controller and class that serve the site are on
[Ingress controllers]({{ '/admin/ingress/' | relative_url }}); what a site's owner writes is on
[HTTPS sites]({{ '/use/ingress/#3--the-certificate' | relative_url }}).

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
