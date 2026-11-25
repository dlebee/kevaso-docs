---
title: Getting started with kvo
permalink: /use/
---

# Getting started with kvo

An administrator runs the cluster and has let you in. This path takes you from there to a silo you
work in: install `kvo`, sign in, make a silo, step inside, then give it ports, a secret, a volume,
and a move. Each step is short and links to the page with the whole story.

You never need SSH to a machine, nor `kvoadm`: everything here is `kvo`, with the access you were
given.

---

## What you start with

An administrator invited you to a **namespace** and sent you two things:

- a **`kvo login …` line**, with the cluster's address and its CA fingerprint;
- a **one-time code**, good for 15 minutes.

You are most likely an `editor` there: you make and change silos and what goes with them. A few
things stay with an administrator (namespaces, roles, opening a port to the outside); the pages say
where. A script or CI job gets a token instead of an invite ([In CI]({{ '/use/sign-in/#in-ci' | relative_url }})).

---

## 1 · Install kvo

`kvo` is one program for macOS, Linux and Windows:
[Install kvo]({{ '/use/install/' | relative_url }}).

## 2 · Sign in

Run the line you were sent. It looks like this:

```bash
kvo login https://203.0.113.10:6443 --ca sha256:4b1d…
```

Your browser opens; paste the code. Then make your namespace the default, so you can leave out
`-n`:

```bash
kvo ns payments
```

```bash
kvo auth whoami
```

More, and how to keep several clusters: [Signing in and contexts]({{ '/use/sign-in/' | relative_url }}).

## 3 · Your first silo

A silo is a YAML object. Save this as `hello.yaml`, with your namespace in place of `payments`:

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: hello, namespace: payments }
spec:
  runtime: cloud-hypervisor
  vm: { image: ubuntu-24.04 }
```

```bash
kvo apply -f hello.yaml
```

```bash
kvo watch silos
```

When it says `phase=Running`, press Ctrl-C. That is a microVM with its own kernel and a 20 GiB disk
that stays. Other runtimes (containers, OpenShell sandboxes, macOS) are on
[Runtimes]({{ '/use/silos/runtimes/' | relative_url }}); init steps and the fields are on
[Your first silo]({{ '/use/silos/first-silo/' | relative_url }}).

## 4 · Step inside

Run one command:

```bash
kvo exec silo hello -- uname -a
```

Get a shell:

```bash
kvo ssh silo hello
```

Copy a file in:

```bash
kvo cp ./notes.txt silo/hello:/root/
```

None of these is SSH: they go through the worker's own channel, are checked against your role, and
are written to the audit log.
[Terminals, files and the small UI]({{ '/use/silos/terminals/' | relative_url }}).

## 5 · Open a port

Nothing can reach a silo until it says so. It **declares** its ports, and its **service** opens
each one to the silos you name. This lets the silo `client` connect to port 8080:

```yaml
spec:
  ports: [{ name: web, port: 8080 }]
  service:
    ports: [{ port: web, allowFrom: [client] }]
```

The silo now has a name too, `hello.payments.kevaso`. Opening a port on a host's address, to the
outside, is an administrator's.
[Addresses, ports and names]({{ '/use/networking/model/' | relative_url }}),
[Networking]({{ '/use/networking/' | relative_url }}),
[Port leases]({{ '/use/networking/leases/' | relative_url }}) for ports that open only for a while.

## 6 · Give it a secret

Store a value. `--from-file -` reads it from stdin, so it never sits on the command line:

```bash
kvo create secret api-key --from-file -
```

A silo gets a secret only through a **Grant**, which also says how. This one adds the key to the
silo's requests to one API, outside the silo, so the silo never holds it:

```yaml
apiVersion: kevaso/v1
kind: Grant
metadata: { name: api-access, namespace: payments }
spec:
  secrets:
    - name: api-key
      deliver: inject
      to: { host: api.example.com, header: x-api-key, format: "{% raw %}{{value}}{% endraw %}" }
```

The silo names the grant in `spec.grants: [api-access]`. A Grant is also what lets a silo reach
anything outside the cluster. [Secrets]({{ '/use/secrets/' | relative_url }}).

## 7 · Add a volume

A volume is kept data with a size and a place inside the silo:

```yaml
spec:
  volumes:
    - { name: data, class: local, size: 50Gi, mountPath: /data }
```

Which classes there are is the administrator's choice: `local` stays on the host, others can be
reached from every host. [Volumes]({{ '/use/silos/volumes/' | relative_url }}).

## 8 · Move it

A silo can move to another host with its data when its disk and volumes are on a class other hosts
can reach. `hello` keeps its disk on `local`, the default, so it stays on its host. The `MOVABLE`
column says where each silo could go:

```bash
kvo get silos -o wide
```

A silo that can move goes with:

```bash
kvo move silo NAME --to HOST
```

How to make a microVM movable, and what a container does instead:
[Moves]({{ '/use/silos/moves/' | relative_url }}).

## Clean up

```bash
kvo delete silo hello
```

---

## Where next

| If you want to | Go to |
|---|---|
| Run many copies, update an image | [Sets, images and updates]({{ '/use/silos/sets/' | relative_url }}) |
| Send traffic only to silos that are ready | [Probes and routing]({{ '/use/silos/probes/' | relative_url }}) |
| Keep a program running inside | [Daemons and upkeep]({{ '/use/silos/daemons/' | relative_url }}) |
| Show a status to the people watching | [Panes]({{ '/use/panes/' | relative_url }}) |
| Serve a site over HTTPS | [HTTPS sites]({{ '/use/ingress/' | relative_url }}) |
| Add commands to `kvo` | [kvo plugins]({{ '/use/kvo-plugins/' | relative_url }}) |
| Know what to ask an administrator for | [What an admin does for you first]({{ '/use/sign-in/#what-an-admin-does-for-you-first' | relative_url }}) |
| Look up a command | [kvo commands]({{ '/reference/kvo/' | relative_url }}) |
