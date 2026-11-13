---
title: Terminals, files and the small UI
permalink: /silos/terminals/
---

# Terminals, files and the small UI

Four ways in, all through the same channel as `kvo exec`: your `kvo` talks to the control plane,
which asks the silo's host to run the command inside the silo. No SSH server runs in the silo, and
Kevaso installs nothing in it.

| You want | Run |
|---|---|
| One command | `kvo exec silo NAME -- CMD` |
| A command on a real terminal (an editor, a REPL, `top`) | `kvo exec -it silo NAME -- CMD` |
| A shell | `kvo ssh silo NAME` |
| Move files | `kvo cp SRC DST` |
| A terminal and a file browser in a web page | `kvo ui silo NAME` |

All of them take `-n NAMESPACE` like the other `kvo` commands.

---

## A terminal

```bash
kvo exec -it silo api -n payments -- top
```

`-t` gives the command a terminal (colours, line editing, full-screen programs). It follows your
window when you resize it, and the command's exit code is `kvo`'s. `-i` is accepted so docker and
kubectl habits work, but standard input always reaches the command.

```bash
kvo ssh silo api -n payments
```

`kvo ssh` is a login shell: `bash -l` if the image has bash, otherwise `sh -l`. It sets `TERM` to
`xterm-256color` if your terminal didn't say. The name is only a habit: **there is no SSH
here.** It's the same channel, so it needs no open port and no key in the silo. To run one command
on a terminal instead of a shell, use `kvo exec -it`.

**When you leave** (you close the window, `kvo` is killed, your network drops), Kevaso stops the
command in the silo too, so it isn't left running with no one to talk to.

What was checked, and what wasn't:

| Runtime | Checked on a real machine? |
|---|---|
| `docker`, `podman` | Yes, on real Linux (Alpine, Ubuntu and Debian containers) |
| `cloud-hypervisor` | In tests with a simulated guest only. Not yet on a real VM. |
| `openshell`, `openshell-container` | In tests only. Not yet against a real OpenShell gateway: it isn't known whether its CLI passes window resizes or takes a terminal. |

### When your client drops

If `kvo exec` or `kvo ssh` is stopped (Ctrl-C, a closed laptop, a lost connection), Kevaso stops
what it started in the silo: the command, everything it started, and every process in the session
it led. That holds even if the command clears its own environment (`env -i …`) or forks while
Kevaso is stopping it.

What can still **escape**: a program that leaves the tree and the session on purpose (a daemon:
fork, `setsid`, exit the parent) and carries none of Kevaso's marker (`KEVASO_EXEC`) in its
environment. For something meant to outlive you, use a [daemon]({{ '/silos/daemons/' | relative_url }}).

On a microVM, `kvo exec` runs the command through `sudo` in the guest, so the whole command line
lands in the guest's journal. **Don't put a secret on a command line.**

---

## Copy files

Name the silo side as `silo/NAME:/ABSOLUTE/PATH`. The other side is a path on your computer.

```bash
kvo cp ./app.conf silo/api:/etc/ -n payments
```

```bash
kvo cp silo/api:/var/log ./logs -n payments
```

It works like `cp`: if the destination is an existing folder, the source goes inside it; otherwise
the destination is the new name. Folders are copied whole. When it's done it prints what moved:

```text
silo/api:/etc/app.conf: 1 files, 0 folders, 0 links, 612 bytes
```

| | |
|---|---|
| `--max-size SIZE` | The most one copy may move, like `500MiB` or `2GiB`. Default `1GiB`; the control plane stops at 4 GiB whatever you ask. |

- **Anything it can't copy is skipped**, and named on standard error.
- **Unpacking on your computer can't write outside the destination** you gave.
- Paths in the silo must be absolute. One of the two sides must be a silo, not both.

---

## The small UI

```bash
kvo ui silo api -n payments
```

```text
silo payments/api: http://127.0.0.1:52817/?token=…
(the link works once, from this machine; Ctrl-C stops it)
```

It opens your browser on a page for that one silo, with:

- **A terminal**, the same as `kvo ssh`, in the page.
- **A file browser**: look through folders, download a file (a folder downloads as a `.tar`) and
  upload one.

It stops when you press Ctrl-C. `--no-browser` prints the link and doesn't open one. The same
`--max-size` limit applies to uploads and downloads.

**Not done yet:**

- The file browser can't delete, rename or make folders. Use the terminal.
- Names containing a newline aren't listed.
- The page has no view of the silo's screen, no graphical desktop. That is later.

### Is it safe to leave running?

It's built so that only you, in the browser you opened, can use it:

- **It listens on `127.0.0.1` only**, on a random port. Nothing on your network can reach it.
- **The link works once.** The first visit swaps the token in it for a session cookie (`HttpOnly`,
  `SameSite=Strict`, named after the port) and the token is dead. Run `kvo ui` again for a new link.
- **Other websites can't drive it.** A request has to name `127.0.0.1:PORT` as its host (this
  stops DNS-rebinding tricks), and anything that changes something, and every terminal connection,
  must come from the page's own origin. Requests from other sites are refused.
- **The page never holds your sign-in.** `kvo` does. The page can only use this one silo's terminal
  and files, as you.
- **It's only as safe as your computer.** Anyone who can use your browser or your user account while
  it runs has a shell in the silo. Stop it when you're done.

---

## Who may do it

Each of these is its own permission, checked and written to the [audit log]({{ '/operations/' | relative_url }}#the-audit-log):

| Command | Needs |
|---|---|
| `kvo exec` (with or without `-it`) | `create` on `silos/exec` |
| `kvo ssh`, the terminal in `kvo ui` | `create` on `silos/ssh` |
| `kvo cp` out of a silo, the file browser's list and download | `get` on `silos/files` |
| `kvo cp` into a silo, the file browser's upload | `create` on `silos/files` |

The built-in `editor` and `admin` roles have all of them; `viewer` has none. They're as strong as
`exec`: anyone with one can read what the silo can read, so give them as carefully. See
[People and access]({{ '/access/' | relative_url }}#roles).

Check yours:

```bash
kvo auth can-i create silos/ssh -n payments
```

---

## What the image needs

The files and the terminal use what's already in the image:

| Needed | For |
|---|---|
| `/bin/sh` | `kvo ssh` and the terminal in `kvo ui` |
| `bash` | Optional: `kvo ssh` uses it when it's there |
| `tar` | `kvo cp` and the file browser, both ways |
| `ls` (GNU or BusyBox) | Listing files in the file browser, and checking a destination in `kvo cp` |

Container images built from scratch, or "distroless" ones, have none of these, so none of these
commands work in them. Debug such a container from its host instead.
