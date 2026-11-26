---
title: Runtimes
permalink: /use/silos/runtimes/
redirect_from: /silos/runtimes/
---

# Runtimes

A silo is always the same thing to you (create, exec, its service, its secrets); the **runtime** is
how it's made. Pick one per silo with `spec.runtime`.

---

## At a glance

| | `cloud-hypervisor` | `openshell` | `openshell-container` | `docker` / `podman` |
|---|---|---|---|---|
| **Isolation** | MicroVM, own kernel | MicroVM (OpenShell) | Container (OpenShell) | Container |
| **Needs KVM** | Yes | Yes | No | No |
| **Network** | Real network card: TCP and UDP, in and out | None: outbound TCP through OpenShell's policy only | None: same | Its own namespace on the silo bridge: TCP and UDP |
| **Inbound services** | Yes, client IP kept | No | No | Yes, client IP kept |
| **Persists** | Its own disk, and [volumes]({{ '/use/silos/volumes/' | relative_url }}) | `/sandbox` | `/sandbox` | Volumes |
| **Injection** | Kevaso's egress gate | OpenShell (placeholder in an env var) | OpenShell | Kevaso's egress gate |
| **Local endpoint** (`169.254.42.1`) | Yes | Not yet | Not yet | Yes |
| **Root inside** | Yes | No (a sandbox user) | No | The image's user |
| **Good for** | Dev boxes, servers, Docker inside | Untrusted code | The same, on hosts without KVM | Apps packaged as images |

A runtime with no inbound network (the OpenShell ones) has no way to take a connection, so don't
open ports on its silos. Admission refuses a grant that an OpenShell silo can't express (an address
range, for example).

A Mac worker has one more runtime, `tart`: macOS silos ([below](#tart-macos-silos)).

What a silo starts from depends on the runtime: `vm:` for `cloud-hypervisor`, `container:` for
the rest ([what a silo starts from]({{ '/use/silos/sets/#what-a-silo-starts-from' | relative_url }})).

---

## cloud-hypervisor

The default choice: a real Linux machine, booted from `ubuntu-24.04`, with its own disk.

- It boots from `vm: { image: … }` (a built-in OS image or a VMImage) or `vm: { disk: { url, sha256 } }`.
  A download of that URL is capped at `disk.size`, or at the default 20 GiB disk when `disk.size`
  is omitted, and a redirect to a private, loopback, link-local or cluster address is refused.
  The messages are on [what a silo starts from]({{ '/use/silos/sets/#vm' | relative_url }}).
  A disk image is accepted only when it is raw, or a plain qcow2: version 2 or 3, no backing file,
  no external data file and no encryption. Anything else is refused, and the silo does not start:

```text
/var/lib/kevaso/images/…: the image names a backing file: only a raw disk, or a qcow2 with no backing file, no external data file and no encryption, can be a silo's disk
```

  The same sentence names an external data file, encryption, a truncated header, or another format
  (`VMDK is not a silo disk`, and the same for VHD, VHDX, VDI, QED and a Parallels disk).
- The hypervisor process runs as its own unprivileged user, with no capabilities, a system-call
  filter and only its own disk and network device. It sees only its own runtime directory under
  `/run`.
- Its serial log is capped at about 10 MiB, plus one older file, `serial.log.1`.
- A finished create or reset whose unit file disappears is not created again over its disk. Delete
  the silo to start over. A create that was killed before its unit was written is still retried.
- Init steps run as root inside, through SSH from the host only (a key the host holds; nothing else
  can log in).
- Nothing of Kevaso's is installed in it. The host adds its egress CA to the silo's trust store and
  writes `file` and `env` secrets into it, through the runtime.

---

## openshell

A silo as an [NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell) sandbox: a microVM with **no
network card at all**. Programs inside reach only what OpenShell's policy allows, and keys are
swapped in by OpenShell's proxy. A grant opens a host and a port. An
[OpenShell policy](#openshell-policies) adds a path, read-only or read-write access, REST, and which
programs may use a rule.

```yaml
apiVersion: kevaso/v1
kind: Grant
metadata: { name: api-access, namespace: ops }
spec:
  egress: [{ to: api.example.com }]
  secrets:
    - name: api-key
      key: token
      deliver: inject
      env: API_KEY             # the variable holding the placeholder
      to: { host: api.example.com, header: x-api-key, format: "{% raw %}{{value}}{% endraw %}" }
---
apiVersion: kevaso/v1
kind: Silo
metadata: { name: worker-1, namespace: ops }
spec:
  runtime: openshell
  container: { image: ubuntu-24.04 }      # a registry reference; a rootfs tar name is refused
  resources: { cpu: 2, memory: 4Gi }
  grants: [api-access]
```

Inside, `$API_KEY` is a placeholder; a program that sends it as its key reaches the API
with the real one.

| Good to know | |
|---|---|
| Grants | Names and single addresses, TCP only, each port listed (at most 16 per rule) |
| Policies | [OpenShell policies](#openshell-policies), listed in `openshellPolicies` |
| Secrets | `inject` only. `file`, `env`, `broker`, `sign` and approval are refused for these runtimes, with the reason |
| Injection format | `Bearer {% raw %}{{value}}{% endraw %}` in `Authorization`, or `{% raw %}{{value}}{% endraw %}` in any header. `{% raw %}{{value}}{% endraw %}` is the named key's value, not a key name |
| Init steps | Run as the sandbox's user, not root |
| `kvo exec` stdin | At most 4 MiB (compress bigger files) |
| Gateways | One per namespace by default (`--openshell-gateway-scope host \| namespace \| silo` on the worker): sandboxes of one gateway share its user, so the scope is how far an escape reaches. Each gateway runs as its own user, locked down. |
| `onControlPlaneLoss: Stop` | Refused, on `openshell` and `openshell-container`. The gateway brings the sandbox back by itself, so nothing keeps it stopped. `onControlPlaneLoss: Stop isn't available on openshell: its gateway brings its sandboxes back by itself (a restart of the gateway, a reboot), so nothing keeps it stopped without the control plane` |

## openshell-container

The same as `openshell`, on OpenShell's Docker driver: for hosts without KVM. The workload has no
network; a supervisor container makes the allowed calls. It lists [OpenShell policies](#openshell-policies)
the same way.

---

## OpenShell policies

An **OpenShellPolicy** is a namespaced object for an OpenShell sandbox's own network rules. A grant
opens a host and a port. A policy adds a path, read-only or read-write access, the REST protocol,
and which programs may use a rule.

A silo on `openshell` or `openshell-container` lists policies by name in `spec.openshellPolicies`,
beside `grants`. The names are policies in that silo's namespace. A silo set lists them on
`spec.template.openshellPolicies`. An editor creates, changes and deletes policies in their own
namespace. A viewer can read them.

Kevaso writes the silo's grants and the policies it lists into one policy for the sandbox. The
sandbox is created with that policy. After a restart, Kevaso pushes it again. After a gateway
outage, Kevaso applies it when the sandbox next starts, including a change you applied while the
gateway was down. A path, read-only or read-write access, or REST is always enforced. A request
that breaks the limit is refused.

Change a policy, or the list on the silo, and apply. Kevaso pushes the new policy to the running
sandbox.

### Allow a read-only GET

This policy allows a GET of `https://example.com/`. The silo lists it. Save it as `example.yaml`.

```yaml
apiVersion: kevaso/v1
kind: OpenShellPolicy
metadata: { name: example, namespace: dev }
spec:
  networkPolicies:
    - name: web
      endpoints:
        - host: example.com
          port: 443
          protocol: rest
          path: /
          access: read-only
---
apiVersion: kevaso/v1
kind: Silo
metadata: { name: worker, namespace: dev }
spec:
  runtime: openshell
  container: { image: ubuntu-24.04 }
  openshellPolicies: [example]
```

```bash
kvo apply -f example.yaml
```

`protocol: rest` and `access: read-only` allow that GET. A POST to the same URL is refused. Another
path is refused. Leave `binaries` out and every program in the sandbox may use the rule. List
absolute paths under `binaries`, such as `/usr/bin/curl`, and only those programs may use it.
`access: read-write` allows writing as well as reading.

On a host without KVM, set `runtime: openshell-container`.

A rule has a lower-case name (letters, digits, `-`) and 1 to 16 endpoints. An endpoint has a host
and a port from 1 to 65535. A host is a name or one public IPv4 address. A host name is stored in
lower case. One public address written as a `/32` is stored as that address. `path` is an absolute
path glob. A policy holds 1 to 32 rules. A silo names at most 32 policies.

### With grants

Kevaso puts the grants and the policies in the sandbox's one policy. A grant opens its host and
port. A policy adds its path, its access, its protocol and its programs. Name a grant in `grants`
and a policy in `openshellPolicies` on the same silo. Both stay in force. A grant for
`example.com` opens that host and port, POST included. The policy's rule allows the GET as well.
To allow only that GET, list the policy and leave `example.com` out of `grants`.

The policies follow the order of `openshellPolicies`. Each policy's rules and endpoints stay in the
order you wrote them.

A silo that lists neither a grant nor a policy can reach nothing outside.

A name you add has to exist in the namespace. Apply is refused otherwise:
`silo worker: openshell policy "missing" doesn't exist in namespace dev`. A silo set says `siloset`
where that message says `silo`. Delete a policy the silo still lists, and the next push leaves that
policy out. The name may stay on the silo. `kvo describe silo worker` shows `PolicyApplied`: what
is in force, and what could not be enforced.

### What is refused

- A runtime other than `openshell` or `openshell-container` that sets `openshellPolicies`. The
  message is `openshellPolicies is only for runtime openshell or openshell-container; docker can't use one`
  (it names the runtime). On a silo set the message starts with `template:`.
- A host inside the cluster (a silo address, a service address, or the mesh), or loopback,
  link-local, or a private address. The message names the range:
  `endpoint host "127.0.0.1" is inside 127.0.0.0/8 (loopback), which a policy never opens`.
  A private address is refused with `hosts are names or public addresses`.
- A range, or anything other than a name or one public IPv4 address. Hosts are IPv4 for now.
- A policy with no rules, or more than 32: `an OpenShellPolicy needs 1 to 32 network policies`.
- A rule with no endpoints, or more than 16. More than 16 programs on one rule.
- Two rules with the same name, or a rule name other than a lower-case name (letters, digits, `-`).
- A silo that names more than 32 policies, the same name twice, or a name other than a lower-case
  name (letters, digits, `-`).
- `protocol` other than `rest`. `access` other than `read-only` or `read-write`.
- A path that does not start with `/`, that has a `..` segment or a control character, or that is
  longer than 200 characters:
  `path "../x": an absolute path glob (starts with /), no control character or .. segment, at most 200 characters`.
  A program path has the same limit.
- `binaries` set to an empty list. Leave the field out for every program, or list 1 to 16 paths.
- A field the kind doesn't have.
- A policy name that is not in the namespace, when you add it to a silo or a silo set.

### Remove a policy

Take the name out of `openshellPolicies` and apply the silo again. Kevaso pushes the sandbox policy
without that rule.

To delete the object:

```bash
kvo delete openshellpolicy example -n dev
```

A silo that still lists the name loses that rule on the next push. Its grants and its other
policies stay. With no grant and no policy left, the sandbox can reach nothing outside.

---

## docker and podman

A container from any image, on the **same silo bridge as the VMs**, so everything on
[Addresses, ports and names]({{ '/use/networking/model/' | relative_url }}) and [Networking]({{ '/use/networking/' | relative_url }})
applies unchanged: default deny, grants, `allowFrom`, `hostPort`, the egress gate.

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: web, namespace: shop }
spec:
  runtime: docker
  container:
    image: docker.io/library/python:3.12-slim
    command: [python3, -m, http.server, "8080", --directory, /data]
  volumes: [{ name: data, mountPath: /data }]
  ports: [{ name: http, port: 8080 }]
  service:
    ports: [{ port: http, hostPort: [internal] }]   # opened on the host's internal IP
  init:
    steps:
      - name: seed
        run: echo hello > /data/index.html
```

Opening a port on a host address (`hostPort`) is a namespace admin's (`create` on
`services/expose-public`, [People and access]({{ '/admin/access/#roles' | relative_url }})): a plain
`editor` gets `may not expose silos publicly`. Ask an admin to apply this one, or leave out
`service` and reach it from other silos with `allowFrom`. Once it is applied,
`curl http://HOST_INTERNAL_IP:8080/` answers `hello`.

| Good to know | |
|---|---|
| `container.image` | A registry reference; `ubuntu-24.04` means `docker.io/library/ubuntu:24.04`. A private registry takes `container.pull` ([Secrets]({{ '/use/secrets/' | relative_url }})). On `openshell` and `openshell-container` a rootfs archive name (`.tar`, `.tar.gz`, `.tgz`) is refused: `container.image is a registry reference, not a rootfs archive (.tar, .tar.gz, .tgz)` |
| `container.command` | The main process. Without it, the image's own (for `ubuntu-24.04`, one that sleeps, so it's a machine to `exec` into) |
| Init steps | **Init containers**: each step runs in a container of its own from the silo's image, on its network and volumes, **before** the main container starts |
| Persistence | Volumes survive restarts; the container's own files stay on its host |
| Secrets | `inject`, `broker`, `file` and `env`. `broker` secrets come from the [local endpoint]({{ '/use/secrets/' | relative_url }}), so the image needs `curl` or any HTTP client. `file` secrets are a read-only bind mount of a host tmpfs under `/run/secrets/…`. `env` secrets are set by Kevaso's own container runtime on Docker, so `docker inspect` and Docker's disk don't hold them; where that isn't available (podman, rootless Docker) they go through `--env-file` and admission warns. Nothing of Kevaso's is put in your image |
| Volume `size` | Not enforced on containers |
| Capabilities | `MKNOD`, `NET_RAW` and `SYS_CHROOT` are dropped. The container's root is the host's uid 0. There is no way to add those three back yet |
| Logs | Three files of 10 MB (`json-file`, `max-size=10m`, `max-file=3`) |

A container silo cannot ping. Ping opens a raw socket, and `NET_RAW` is dropped, so the kernel
refuses it. The firewall would allow ping to this host's gateway from a microVM; a container
never sends it. A workload that needs `MKNOD`, `NET_RAW` or `SYS_CHROOT` cannot have them yet.

---

## tart (macOS silos)

On a [Mac worker]({{ '/admin/workers/mac/' | relative_url }}), `runtime: tart` runs a macOS VM through
[Tart](https://tart.run), from an OCI image:

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: mac, namespace: demo }
spec:
  runtime: tart
  container: { image: ghcr.io/cirruslabs/macos-tahoe-base:latest }
  resources: { cpu: 4, memory: 8Gi }
```

| Good to know | |
|---|---|
| How many | At most **two** running per Mac: Apple's licence for macOS allows two virtual copies. A third waits, `Pending`, until one stops |
| Size | 4 CPUs and 8 GiB by default. The image is large (25 to 70 GB): pull it on the Mac first (`tart pull IMAGE`), or the first start takes that long |
| Network | Through `softnet`: the internet, yes; the Mac's LAN, the other silos and the cluster, no. It has **no silo IP**, and needs no grant to reach the internet |
| Refused when you apply | `ports` and `service` (nothing can reach it), `grants` (the egress gate is a Linux host's), and volumes: its disk is the VM itself, kept on its Mac |
| `kvo exec` | Works, as on any silo (`kvo exec silo mac -n demo -- sw_vers`) |
| Tart | Licensed FSL-1.1-ALv2; Kevaso only runs the `tart` program installed on the Mac |

