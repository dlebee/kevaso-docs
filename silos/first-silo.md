---
title: Your first silo
permalink: /silos/first-silo/
---

# Your first silo

You have a cluster with a worker ([Installing]({{ '/setup/prepare/' | relative_url }})). Now make a microVM, watch it come
up, and step inside.

---

## The smallest silo

A silo needs nothing ensured inside it. Init steps, [daemons and upkeep]({{ '/silos/daemons/' | relative_url }})
and a [readiness check]({{ '/silos/sets/' | relative_url }}#readiness) are all optional. A silo with none of them is a normal,
stateful machine: Kevaso creates it, keeps it started, and otherwise leaves it alone. This is the
whole manifest:

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: plain, namespace: demo }
spec:
  runtime: cloud-hypervisor
  image: ubuntu-24.04
```

With no `resources` or `disk`, a microVM gets 1 CPU, 1 GiB of memory and a 20 GiB disk. With no
`init` it is `Running` and `Ready` as soon as it has booted. Whatever you put on its disk stays
there across stops and restarts. The rest of this page adds an init step to that, as an example.

---

## Describe it

Save this as `hello.yaml`:

```yaml
apiVersion: kevaso/v1
kind: Namespace
metadata: { name: demo }
---
apiVersion: kevaso/v1
kind: Silo
metadata: { name: hello, namespace: demo }
spec:
  runtime: cloud-hypervisor        # a microVM with its own kernel and network card
  image: ubuntu-24.04
  resources: { cpu: 1, memory: 1Gi }
  disk: { size: 10Gi }             # its own disk, kept for its whole life
  init:                            # optional: run inside, in order, before it's Ready
    steps:
      - name: hello
        run: echo "hello from $(hostname)" > /root/hello
      - name: python
        shell: python3             # any interpreter in the image
        run: |
          import platform; print("kernel", platform.release())
```

| Field | Means |
|---|---|
| `runtime` | What kind of silo: here a microVM. Others in [Runtimes]({{ '/silos/runtimes/' | relative_url }}). |
| `image` | What it boots. `ubuntu-24.04` is built in (downloaded once per host, digest-checked). |
| `resources` | CPUs and memory it may use. The scheduler only places it where they fit. |
| `disk` | Its root disk. It lives on the host it was created on. |
| `init.steps` | Optional. Scripts, run as root inside the silo, in order. If you give any, the silo is Ready only after the last one succeeds. |

---

## Create it

```bash
bin/kvo apply -f hello.yaml
```

```bash
bin/kvo watch silos -n demo
```

```
NOW  silo/demo/hello  phase=Pending  host=-
NOW  silo/demo/hello  phase=Scheduled  host=hv1
NOW  silo/demo/hello  phase=Starting  host=hv1  Ready: creating on hv1
NOW  silo/demo/hello  phase=Starting  host=hv1  init=0/2  Ready: init step hello
NOW  silo/demo/hello  phase=Running  host=hv1  init=2/2  address=10.64.1.2
```

The first silo on a host takes a few minutes (the image is copied onto its disk); later ones are
faster. Ctrl-C stops watching, not the silo.

---

## Look at it

```bash
bin/kvo get silos -n demo
```

```bash
bin/kvo describe silo hello -n demo
```

`describe` shows everything about it: where it runs, its address, its init steps with their exit
codes and durations, and its conditions (the *why* when something isn't Ready).

---

## Step inside

```bash
bin/kvo exec silo hello -n demo -- cat /root/hello
```

```bash
bin/kvo exec silo hello -n demo -- bash -c "uname -a; df -h /"
```

`exec` runs one command and exits with its exit code, so it works in scripts. It goes through the
worker's dial-out channel: no SSH, no open port on the host, and every call is checked against your
permissions and written to the audit log.

> **Is it locked down?** Each VM's hypervisor runs as its own unprivileged user, with no
> capabilities, a system-call filter, and access to only its own disk and network device.

---

## When an init step fails

The silo stays up (so you can look inside) but never becomes Ready, and `describe` shows which step
failed and the end of its output:

```
Conditions:
  TYPE        STATUS  REASON      MESSAGE
  InitFailed  True    StepFailed  step python exited 1: …
```

Fix the step in `hello.yaml` and `apply` again.

---

## Next

- Get a shell, or move files: [Terminals, files and the small UI]({{ '/silos/terminals/' | relative_url }}).
- Keep a program running, or a check passing, if you want that (a silo doesn't need it): [Daemons and upkeep]({{ '/silos/daemons/' | relative_url }}).
- Show a status to the people watching: [Panes]({{ '/panes/' | relative_url }}).

---

## Clean up

```bash
bin/kvo delete silo hello -n demo
```

The VM, its disk and its network device go away on the host.
