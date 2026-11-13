---
title: Daemons and upkeep
permalink: /silos/daemons/
---

# Inside a silo: daemons and upkeep

Two lists in a silo's spec keep the inside of it the way you want:

| List | What it is | Like |
|---|---|---|
| `daemons` | A process that must keep running. If it dies, it's started again. | A tiny process supervisor |
| `upkeep` | A **check** that runs now and then, and a **fix** that runs only when the check fails. | A cron job that looks first |

Both run after the silo's [init steps]({{ '/silos/first-silo/' | relative_url }}) are done, for as
long as the silo is running. Nothing of Kevaso's is installed in the silo: the host's agent starts
and checks them by running ordinary commands inside it, the same way `kvo exec` does.

---

## Daemons

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: api, namespace: payments }
spec:
  runtime: docker
  image: ghcr.io/acme/api:1.4
  daemons:
    - name: web
      run: /usr/local/bin/api --port 8080
    - name: worker
      run: exec /usr/local/bin/api-worker
      user: app
      restart: onFailure
```

| Field | Meaning | Default |
|---|---|---|
| `name` | Lower-case letters, digits and `-`. Unique in the silo. | required |
| `run` | A script, run as `SHELL -c 'SCRIPT'` | required |
| `shell` | The shell to run it with | `sh` |
| `user` | Run as this user | `root` |
| `restart` | `always`, `onFailure` or `never` | `always` |

A daemon's output (standard output and error) goes to `/run/kevaso/daemons/NAME.log` inside the
silo. Read it with `kvo exec`:

```bash
kvo exec silo api -n payments -- tail -n 50 /run/kevaso/daemons/web.log
```

### What the restart policies do

| `restart` | When it ends with exit code 0 | When it ends any other way |
|---|---|---|
| `always` | Started again | Started again |
| `onFailure` | Stays stopped (`Exited`) | Started again |
| `never` | Stays stopped (`Exited`) | Stays stopped (`Exited`) |

A daemon that keeps dying isn't restarted in a tight loop. The wait before each restart doubles:
10 seconds, 20, 40, and so on, up to 5 minutes. A daemon that ran for 10 minutes before dying
starts over at 10 seconds. A running daemon is looked at every 15 seconds.

In a microVM, the daemons' records are kept in `/run`, which is memory: after a reboot every daemon
is started again. What a container restart does to them is not confirmed yet.

### Taking one out

Remove it from the list and apply. Kevaso stops it (the whole process group, so its children too)
the next time it looks.

**Not done yet:** changing a running daemon's `run`, `user` or `shell` doesn't restart it. To
start it with the new settings, take it out of the list, apply, put it back and apply again, or
[reset the silo]({{ '/reference/' | relative_url }}#silos).

---

## Upkeep

```yaml
spec:
  upkeep:
    - name: stale-lock
      every: 6h
      check: test ! -e /data/tmp/stale.lock
      fix: rm -f /data/tmp/stale.lock
    - name: certs
      every: 24h
      check: test -s /etc/ssl/app.pem
      fix: /usr/local/bin/renew-certs
      user: app
```

| Field | Meaning | Default |
|---|---|---|
| `name` | Unique in the silo | required |
| `every` | How often the check runs: a duration like `30m` or `6h`, **at least `1m`** | required |
| `check` | A script. Exit code 0 means fine. | required |
| `fix` | A script, run only when the check fails | required |
| `shell`, `user` | As for daemons | `sh`, `root` |

Each time a task is due:

1. The **check** runs. Exit code 0: the task is `OK`, and nothing else happens.
2. If it fails, the **fix** runs.
3. If the fix exits 0, the **check runs again**. Passing now: the task is `Fixed`.
4. Anything else (the fix failed, or the check still fails after it) makes the task `Failed`, and
   the silo `Degraded`.

A run (check, fix and check again) is stopped after 15 minutes. The first check runs as soon as init is
done; after that, `every` after the last one.

### `Degraded`

A silo with a `Failed` upkeep task gets the condition `Degraded` with the reason `UpkeepFailed` and
the failing tasks' names. It clears by itself, with the reason `UpkeepOK`, the next time every task
passes.

`Degraded` is a warning. It doesn't stop the silo, change its phase or make it not `Ready`, and
Kevaso doesn't restart or move a silo because of it.

---

## What you see

```bash
kvo describe silo api -n payments
```

```text
Daemons:
  NAME    STATE    RESTARTS  LAST EXIT
  web     Running  0         -
  worker  Waiting  3         1
Upkeep:
  NAME        EVERY  STATE   LAST CHECK  MESSAGE
  stale-lock  6h     Fixed   2h ago      -
  certs       24h    Failed  3h ago      renew-certs: cannot reach the CA
Conditions:
  TYPE      STATUS  REASON        SINCE  MESSAGE
  Degraded  True    UpkeepFailed  3h     upkeep failed: certs
```

| Daemon state | Means |
|---|---|
| `Pending` | Not looked at yet |
| `Running` | Up |
| `Waiting` | Died, will be started again after its backoff |
| `Exited` | Died, and its restart policy says to leave it |

`LAST EXIT` is how it last ended. For upkeep, `Pending` means the first check hasn't run yet, and
`MESSAGE` is the end of what the failing script printed (about 300 characters).

---

## What the image needs

Kevaso adds nothing to a silo, so the image has to have what the commands use:

| Needed | For |
|---|---|
| `sh` | Everything. Daemons and upkeep start and probe through `sh`. |
| `setsid` | Optional. With it, a daemon gets its own process group and stopping it takes its children too. Without it, daemons still run; a stop is less thorough. |
| `su` | Only when `user` isn't `root`. |
| The shell you name in `shell` | Only when you set one (`bash`, say). |

Most images have `sh`, and `setsid` comes with util-linux. An image with no shell can't use
daemons or upkeep.
