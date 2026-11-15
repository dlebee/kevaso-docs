---
title: Secrets
permalink: /secrets/
---

# Secrets

The goal: a silo can **use** an API key, a password or a signing key, while having as little of it
as possible inside. Often that's nothing at all.

---

## The four ways, best first

| | Tier | The silo gets | Use it for |
|---|---|---|---|
| 1 | **Injection** (`deliver: inject`) | Nothing. The key is added to its requests outside the silo. | Calls to an HTTP API with a key |
| 2 | **Operations** (`operations.sign`) | Signatures. The key stays in Kevaso. | Signing keys, code signing |
| 3 | **On demand** (`deliver: broker`) | The value, when it asks, from [a local address only it can reach](#the-local-endpoint) | Passwords, anything that can't be injected |
| 4 | **As a file** (`deliver: file`) | A file in memory, written by the host and replaced on rotation | Apps that read a key file |
| — | `deliver: env` | An environment variable. Leaks into child processes and logs: legacy apps only. | Last resort |

**Nothing of Kevaso's is installed in a silo.** Its image needs no helper. Injection happens outside
the silo, files and variables are written by the host, and everything else is plain HTTP to a local
address. An image needs `curl` (or any HTTP client) only for on-demand secrets, signatures and port
leases.

---

## Store a secret

Values come from a file or from what you type, **never from the command line** (other users of the
machine and your shell history would see it). With `--from-file -`, `kvo` reads the value from
your terminal:

```bash
kvo create secret api-key -n dev --from-file - --allow inject --write-only
```

Paste the key, press Enter, then Ctrl-D. Or keep it in a file only you can read and give its path:

```bash
kvo create secret api-key -n dev \
    --from-file ~/keys/api-key.txt \
    --allow inject \
    --write-only
```

| Flag | Meaning |
|---|---|
| `--from-file F` / `--from-file -` | The value, from a file or stdin |
| `--generate` | Kevaso makes it (32 random bytes; or a new private key with `--type`) |
| `--type ed25519` / `ecdsa-p256` | A signing key: only ever used through `sign`, never readable |
| `--allow inject,broker` | The only ways grants may deliver it, whatever they ask |
| `--write-only` | Nobody can read it back, admins included |
| `-l sensitivity=critical` | Labels: roles can give access by label |

The API never returns a value in `get`, `list` or `watch`. Someone with `read-value` can read a
readable secret:

```bash
kvo get secret db-password -n dev --reveal
```

---

## Give it to a silo: a Grant

```yaml
apiVersion: kevaso/v1
kind: Grant
metadata: { name: api-access, namespace: dev }
spec:
  secrets:
    - name: api-key
      deliver: inject
      to: { host: api.example.com, header: x-api-key, format: "{{value}}" }
    - name: db-password                  # deliver: broker is the default
    - name: tls-key
      deliver: file
      path: /run/secrets/tls.pem          # under /run: memory only
    - name: license
      deliver: env                        # written by the host; the variable is LICENSE
  operations:
    sign: { keyRef: signing-key, slashingProtection: true }
---
apiVersion: kevaso/v1
kind: Silo
metadata: { name: worker, namespace: dev }
spec:
  runtime: cloud-hypervisor
  vm: { image: ubuntu-24.04 }
  grants: [api-access]
```

```bash
kvo describe grant api-access -n dev
```

`describe` says which tier each secret uses.

---

## Injection, in detail

For `cloud-hypervisor` and container silos, the host's **egress gate** sits in front of the silo's
HTTPS to `api.example.com`:

1. The silo's TLS is ended with the host's own egress CA, which Kevaso put in the silo's trust
   store. Only names a grant injects for get this; anything else is refused.
2. Each request must be for that same host (no domain fronting).
3. Whatever the silo sent in that header is **removed**, and the secret is set.
4. The gate connects to the real `api.example.com` itself and checks its certificate.

`to.header` defaults to `Authorization` and `to.format` to `Bearer {{value}}`. The program in the silo just calls the API without a key; it works. On **OpenShell** silos the
same grant gives the program an environment variable holding a *placeholder* (`$API_KEY =
openshell:resolve:env:…`), which it sends like a key, and OpenShell swaps the real one in. See
[Runtimes]({{ '/silos/runtimes/' | relative_url }}#openshell).

---

## Files and variables

Kevaso puts a `file` or `env` secret into a silo from the host, and installs nothing inside it. How
depends on the runtime:

| | `file` | `env` |
|---|---|---|
| **MicroVM** (`cloud-hypervisor`) | Written by the host through the runtime, over a pipe, to `path` in the grant (default `/run/secrets/NAME`). Mode `0400`. Rewritten while the silo runs | Written to `/run/kevaso/secrets.env` (`0400`), read by login shells through `/etc/profile.d/kevaso-secrets.sh`. A running process sees no change |
| **Container on Docker** | A folder of the host's memory-only filesystem, mounted read-only into the container. `0400`, owned by root. Replaced in place on rotation, no restart | The container's own environment, set by Kevaso's runtime as the container starts. Not in `docker inspect`, not on Docker's disk. Shown in `SecretsDelivered` |
| **Container on Docker, where Kevaso's runtime can't be used**, and **Podman** | As above | The container's own environment, kept by the engine in the container's config: `inspect` shows it and it is on the host's disk. Admission warns |
| **OpenShell** | Refused, with the reason | Refused, with the reason |

The variable is `env` in the grant, or the secret's name in capitals with `-` as `_`. The value
goes over a pipe or a file in memory: never on a command line.

- **Memory only.** On a microVM the host refuses to write into a folder that isn't a tmpfs (`/run`
  is one on any systemd image). On a container the files come from a folder the host keeps in memory.
- **Where a file can go.** The path must be under `/run/`. On a container it must be in a folder
  under `/run`, like `/run/secrets/NAME`, not in `/run` itself: the host mounts the folder read-only
  and `/run` stays the container's own writable memory. Admission refuses anything else.
- A file in a folder the running container doesn't have mounted yet (a grant added while it runs)
  appears at the container's next start. A microVM gets it at once.
- Values are written before the silo's init steps and again whenever the secret or the grant changes.
  They are removed when no longer delivered, and written again after a reboot or a `kvo reset`.
  Stopping a container silo empties its folder; the host fills it again before the next start.
- A microVM's image needs a POSIX `sh` and `cat`, `mkdir`, `chmod`, `mv` and `rm`. One without them
  gets the silo condition `SecretsDelivered=False` with the reason. A container's image needs
  nothing, so a minimal image with no shell works.
- Init steps of a container run before the main container starts and don't see `file` or `env`
  secrets. A microVM's init steps do.
- A program that keeps a `file` secret open must re-read it on its own schedule: nothing tells it
  the secret changed.
- Values of `env` must be text without a newline. On Podman a value that starts with a quote is
  refused. Deliver those as a file.

### Environment variables on a container

Kevaso's runtime is installed on a Docker host by `kvoadm`. It sits between Docker and its usual
runtime, adds the variables to the container just as it starts, and passes everything else through.
It is used on a rootful Docker, version 24 or later, whose default runtime is `runc`. A container
without an `env` secret never goes through it.

| Who | Can see a variable's value |
|---|---|
| Someone with access to the host's Docker, through Kevaso's runtime | No: not in `docker inspect`, not in the engine's files on disk |
| Someone with access to the host's Docker, on the fallback | Yes: `docker inspect` and `podman inspect` show it, and the engine stores it on the host's disk |
| A process inside the same silo | Yes: it can read its own environment and its parent's |
| `root` on the host | Yes, always: it can read any process's environment and the memory the files live in |

- A container silo's `docker exec` and `kvo exec` shells don't get the variables on Kevaso's
  runtime: Docker makes an exec's environment from its own config. A program the silo starts does
  get them. Read one from the main process with `tr '\0' '\n' < /proc/1/environ`, or deliver the
  secret as a file.
- **The fallback and its warning.** On Podman, a rootless Docker, a default runtime that isn't
  `runc`, a Docker older than 24, or a host where Kevaso's runtime didn't register, the variables go
  in through the engine's own environment option. Applying the silo prints `Warning:` on stderr
  saying the value is delivered as an environment variable, that the engine records it on that
  worker, and that it is visible in the engine's records there (`inspect`, its files on disk) to
  anyone who can read them. Files don't have this weakness. The warning is left out when every ready host of that runtime uses Kevaso's runtime.
  `kvo describe silo` shows the way a silo got in `SecretsDelivered`.
- A rotated variable reaches a container at its next start. Each start reads the new value.
- If the host's Docker remaps users (`userns-remap`), a `0400` file owned by root can't be read
  inside the container. That isn't supported.

A silo that gets an injected secret must trust its host's egress CA. Kevaso arranges that, with no
tool of its own inside:

| Runtime | How |
|---|---|
| Container | The runtime mounts a certificate bundle (the host's CAs plus the egress CA), with `SSL_CERT_FILE` and friends set |
| MicroVM | The host writes the CA to `/etc/kevaso/egress-ca.crt` and runs `update-ca-certificates` (Debian, Ubuntu, Alpine) or `update-ca-trust` (Fedora, RHEL) when the image has one |

An image with neither tool gets a bundle file and environment variables only (`SSL_CERT_FILE`,
`REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS`). Programs that ignore them won't trust
the gate, and `SecretsDelivered` says "CA trusted through environment variables only".

---

## The local endpoint

For on-demand secrets, signatures and port leases, a silo asks plain HTTP at
`http://169.254.42.1`, answered by its own host. It works on `cloud-hypervisor`, `docker` and
`podman` silos, which share the host's silo bridge.

- **The host knows who asks from where the request arrived**, never from what it says. The bridge
  drops a frame whose source address isn't the silo's own, so a silo can't pose as another.
- **A silo has the endpoint only if it needs it:** a grant with an on-demand secret or a `sign`
  operation, or a [leased port]({{ '/networking/leases/' | relative_url }}). Any other silo's packets
  to that address are dropped.
- **Other machines can't reach it.** The host drops it from the mesh and the uplink.
- Every request needs the header `Kevaso-Local: 1`. Requests with an `Origin` or `Sec-Fetch-*`
  header, with another `Host`, or with another path are refused, and so are browser preflights.
  This keeps a web page, or a program tricked into fetching a URL, from asking it by accident.
- Limits: a body only for signing (at most 512 KiB), at most 8 requests at once per silo.

To read a secret:

```bash
curl -fsS -H 'Kevaso-Local: 1' http://169.254.42.1/v1/secrets/db-password
```

To sign a payload (the signature comes back as hex):

```bash
printf 'message 42' | curl -fsS -H 'Kevaso-Local: 1' --data-binary @- 'http://169.254.42.1/v1/sign/signing-key?slot=42'
```

To read the public key (hex):

```bash
curl -fsS -H 'Kevaso-Local: 1' http://169.254.42.1/v1/keys/signing-key
```

To open and close a leased port (see [Port leases]({{ '/networking/leases/' | relative_url }})):

```bash
curl -fsS -H 'Kevaso-Local: 1' -X POST 'http://169.254.42.1/v1/ports/debug/open?for=1h'
```

```bash
curl -fsS -H 'Kevaso-Local: 1' -X POST http://169.254.42.1/v1/ports/debug/close
```

| Route | Answer |
|---|---|
| `GET /v1/secrets/NAME` | The value (`application/octet-stream`); its version in `Kevaso-Version` |
| `POST /v1/sign/KEY[?slot=N]` | The signature, hex. The body is the payload |
| `GET /v1/keys/KEY` | The public key, hex |
| `POST /v1/ports/NAME/open[?for=D]` | `{"port":"debug","until":"…","clamped":false}` |
| `POST /v1/ports/NAME/close` | `{"port":"debug","closed":true}` |

Errors come back as JSON, `{"error": "…"}`; `curl -f` turns a refusal into a failure.

| Status | Meaning |
|---|---|
| `400` | A bad parameter |
| `403` | Refused: a missing header, an approval denied or expired, a lease refused |
| `404` | No such route, or a secret or key that doesn't exist **or isn't this silo's** (the same answer) |
| `405` | Wrong method |
| `409` | Signing refused by slashing protection |
| `413` | Payload over 512 KiB |
| `428` | Waiting for a person's yes: [approvals](#ask-a-person-first-approvals) |
| `429` | Too many requests at once from this silo |
| `503` | The control plane can't be reached: try again |

- Asking for a secret the silo has no grant for answers exactly like a name that doesn't exist.
- A host gets secrets only for silos placed on it.
- If a key must never sign two different payloads for the same slot (a system with equivocation
  penalties, say), set `slashingProtection: true` on `sign`. A second, different payload for a slot
  already signed is refused (`409`), and so is any slot before the last.
- Every call, allowed or refused, is in the audit log.
- **`openshell` and `openshell-container` silos have no endpoint.** `kvo describe` shows the silo
  condition `LocalEndpoint=False NotSupported` for one that has a grant needing it.

> **Proven on one bare-metal machine** (Ubuntu 26.04): the endpoint, files and variables on a
> microVM, and files and variables on Docker through Kevaso's runtime (including that `docker inspect`
> and Docker's folder on disk hold no value). **Not yet proven:** files and variables on Podman
> (a Podman silo has run, opened a port and read a secret from the endpoint), the fallback on a real host, a second machine,
> the failure cases, OpenShell. See
> [What's proven where]({{ '/operations/#whats-proven-where' | relative_url }}).

---

## Ask a person first: approvals

If a key is too powerful to use without a person saying yes (an admin token, a deploy key, a
signing key), require approval. It's optional, per secret in a grant. Add `approval: { required: true }`, and every
use waits for a yes:

```yaml
apiVersion: kevaso/v1
kind: Grant
metadata: { name: deploy-access, namespace: ops }
spec:
  secrets:
    - name: admin-token
      deliver: inject
      to: { host: api.example.com }
      approval: { required: true }
  operations:
    sign: { keyRef: deploy-key, approval: { required: true } }
```

A grant without `approval` works as before, and works for broker, inject and sign. File and env
are refused: once the value sits in the silo, nothing can ask again. OpenShell silos inject by
themselves, so an injection that needs approval works only on `cloud-hypervisor`, `docker` and
`podman` silos.

**In the silo**, a call that needs a yes answers HTTP `428` with the pending approval, and the
silo asks again with `approval=NAME` every couple of seconds until it is answered (`200`, or `403`
if denied or expired). Asking with `approval=NAME` doesn't replace the pending request; asking
without it does:

```bash
curl -fsS -H 'Kevaso-Local: 1' 'http://169.254.42.1/v1/secrets/admin-token?for=30m&wait=10m'
```

```text
curl: (22) The requested URL returned error: 428
```

```bash
curl -fsS -H 'Kevaso-Local: 1' 'http://169.254.42.1/v1/secrets/admin-token?for=30m&approval=NAME'
```

The `428` body has the pending name and says `kvo approve NAME -n ops`; it is also in the
`Kevaso-Approval` header. `for` is how long access lasts once approved: until then, the silo uses
the secret without asking again. `wait` is how long a request may wait for an answer. For an
injection, the request itself waits at the egress gate.

**You** see the queue and answer. `kvo get approvals` and `kvo watch approvals` have a `REQUEST` column. It is the first 32 hex digits of the request's id. The full id is 64 hex digits. It names what this request asks for. A silo that asks again with anything different gets a new id. A yes is for the id you read. That keeps it from landing on a different ask.

```bash
kvo get approvals -n ops
```

On a terminal, `kvo approve` shows the request and asks `approve this request (ID)? [y/N]`. `ID` is those 32 hex digits. The yes is for the request it showed. `y` or `yes` approves it. Anything else prints `not approved` and changes nothing.

```bash
kvo approve db-3f9a1c2e -n ops
```

Off a terminal, name the request. `--request` takes the `REQUEST` column, a longer prefix of the same id, or the full 64 hex digits. A prefix shorter than 32 digits is refused. A shorter one can match two requests. `kvo` completes a prefix from the approval you named, and only when that id starts with what you typed. It then sends the full 64 digits.

```bash
kvo approve db-3f9a1c2e -n ops --request 3f9a1c2e4b7d6a0c8e1f2a3b4c5d6e7f
```

```bash
kvo deny db-3f9a1c2e -n ops --reason "not during the freeze"
```

| `--request` on `kvo approve` | |
|---|---|
| The `REQUEST` column, a longer prefix, or the full 64 hex digits | Accepted. The yes is for that request |
| Fewer than 32 hex digits | `request "…": too short, give at least 32 digits of the REQUEST column` |
| Not hex | `request "…": not a request ID (hex digits, from kvo get approvals)` |
| The pending request no longer starts with it | `approval NAME: the request changed since it was read: no pending request starts with … (it is … now)` |
| The full id is for a different request | `Conflict (409): approval NAME: the request changed since it was read: it is request … now (…, for …); read it again and answer that one` |
| Omitted, and stdin is not a terminal | `approve NAME: say which request with --request ID (the REQUEST column of kvo get approvals)` |

- **Every request expires.** Nobody answers: the call fails, saying it expired.
- **Asking again replaces the pending request**, so the queue never fills with copies.
- **Only people answer**, with the verb `approve` on that secret (the built-in `admin` role has
  it). The silo can't, and neither can a service account's token, whatever its roles.
- **Opening a leased port can need a yes too.** That approval is answered with `approve` on
  `silos/exposure`: see [Port leases]({{ '/networking/leases/' | relative_url }}).
- Every request, replacement and answer is in the audit log, with who answered.
- The control plane sets the limits: `kevasod control-plane --approval-wait 10m
  --approval-max-wait 1h --approval-access 15m --approval-max-access 12h` (these are the defaults).

---

## Private registry credentials

A container silo can pull its image from a private registry. The credential is a secret of type
`registry`: the password or token is the value, and the registry host and user are fields.

```bash
kvo create secret registry-pull -n dev --type registry --server registry.example.com --username ci-bot --from-file -
```

Paste the password or token, press Enter, then Ctrl-D. On this command `--server` is the registry,
not the cluster (`--registry` says the same thing). Add `--default-pull` to make it the namespace's
default for that registry.

| Flag (with `--type registry`) | Meaning |
|---|---|
| `--server HOST` / `--registry HOST` | The registry host, like `registry.example.com` or `ghcr.io`. A pasted `https://…/path` is cut to the host |
| `--username U` | The user to sign in as |
| `--from-file F` / `--from-file -` | The password or token. `--generate` is refused |
| `--default-pull` | Every silo in the namespace that pulls from this registry uses it, unless it names another |

Name it from the silo:

```yaml
spec:
  runtime: docker
  container:
    image: registry.example.com/acme/api@sha256:4b2d8f0e9a7c6b5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d
    pull: { secretRef: registry-pull }
```

- **A registry credential never reaches a silo.** It can't be delivered any way, and it's
  write-only unless you say otherwise. Only the agent of the host the silo is placed on gets it, for
  one pull. It hands it to the container engine in a throwaway file in memory and deletes it after.
- Naming a secret in `container.pull` needs `use` on it, like a grant.
- The pull happens before the old copy stops. A refused pull leaves the silo running as it was and
  sets the reason `ImagePullFailed`.
- `docker` and `podman` silos can use one. OpenShell silos pull through their own gateway, take no
  credential, and refuse `container.pull`.
- `kvo describe silo` shows whether the image is pinned by digest, the digest pulled and the pull
  secret. A tag can move; a digest can't.

> **Not proven on real machines:** pulling with a credential has run in tests, not against a real
> private registry.

---

## Rotation

```bash
kvo secret set api-key -n dev --from-file -
```

Paste the new key, Enter, Ctrl-D.

| Delivered as | What happens |
|---|---|
| `inject` | The next request carries the new value. Nothing changes in any silo. |
| `file` | The file is rewritten in every silo that has it, within seconds. |
| `env` | A microVM: the host rewrites the variables file, and a program sees the new value in its next login shell. A container: the next start |
| `broker` | The next request to the endpoint returns it. |

One secret granted to many silos is renewed once, for all of them.

---

## Who can do what with secrets

Secrets have their own verbs, separate powers:

| Verb | Allows |
|---|---|
| `list` | See that a secret exists (name, labels, version), never its value |
| `use` | Name it in a Grant, i.e. let a silo receive it |
| `read-value` | Read the value as a person |
| `write` | Create, change, rotate, delete |
| `approve` | Answer a silo's request to use it ([approvals](#ask-a-person-first-approvals)) |

Rules can target secrets **by label**, so new secrets are covered automatically:

```yaml
kind: Role
metadata: { name: payments-dev, namespace: payments }
rules:
  - { resources: [secrets], verbs: [list, use], selector: { sensitivity: normal } }
  - { resources: [secrets], verbs: [use], resourceNames: [signing-key] }
  - { resources: [secrets], verbs: [approve], selector: { sensitivity: critical } }
```

**No back door through silos:** writing a Grant needs `use` on every secret it names, and giving a
silo a Grant needs `use` on that Grant's secrets too. Being able to create silos doesn't give you the
namespace's secrets.

---

## The master key and the recovery key

Values are stored with **envelope encryption**: each value has its own data key, and only data keys
are wrapped by the cluster's master key (a root-only file on the system node). A stolen copy of the
store opens nothing without it.

The **recovery key** that `kvoadm init` printed is the spare key to your secrets: the master keys are
sealed to it in the store, and the system node keeps only its public half, so it can't use it
itself. Keep it offline, with your other recovery codes. Nobody needs it day to day. It's what opens
your [backups]({{ '/operations/backup/' | relative_url }}): a backup without it can't be restored.

| Task | Command |
|---|---|
| New master key (re-wraps data keys; the recovery key is unchanged) | `kvoadm kms rotate root@HOST --ssh-key …` |
| The recovery key leaked: new master key and new recovery key | `kvoadm kms rotate root@HOST --ssh-key … --new-recovery-key` |

After `--new-recovery-key`, the old recovery key opens nothing stored from then on. Copies of the
store made before still open with it and hold the values of then: destroy them, and rotate the
secrets that matter.
