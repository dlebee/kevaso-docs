---
title: Panes
permalink: /panes/
---

# Panes

A **pane** is a small status display that a silo puts out for the people who run it: "build 40%
done", "needs you", a little dashboard. The silo reports, Kevaso keeps the latest report, and a
client shows it.

| Kind | What the silo sends | How it's shown |
|---|---|---|
| `status` | A state, a progress number, a title, a message and some label/value fields, all plain text | Drawn by the client itself, as plain text. Safe. |
| `html` | A web page (a *bundle*) and some JSON data pushed to it | Only in the [macOS panes app](#the-macos-panes-app), in a sealed view |

Panes are **off** unless the silo's spec turns them on, and they go **one way**: from the silo to
the people watching. Nothing a pane says, or anyone does to a pane, ever reaches back into the
silo.

---

## Turn them on

```yaml
apiVersion: kevaso/v1
kind: Silo
metadata: { name: builder, namespace: team }
spec:
  runtime: cloud-hypervisor
  image: ubuntu-24.04
  panes:
    enabled: true
    allow: [status, html]
    maxPanes: 4
```

| Field | Meaning | Default |
|---|---|---|
| `enabled` | Panes on for this silo | off |
| `allow` | Which kinds it may report: `status`, `html` | `status` only |
| `maxPanes` | How many panes it may have, 0 to 16 | 4 |

A silo that isn't enabled has every report refused, saying so. A silo past `maxPanes` can still
update the panes it has; it can't make a new one.

---

## What the image needs

Reporting uses `kvo-guest`, the silo's helper for talking to its host.

| Runtime | `kvo-guest` |
|---|---|
| `cloud-hypervisor` | Installed on first boot |
| `docker`, `podman` | Only if the image contains it. Kevaso installs nothing in a container. |
| `openshell`, `openshell-container` | No broker: no panes |

---

## Report a status pane

From inside the silo:

```bash
kvo-guest pane report build --state running --progress 0.4 --title "Release build" --message "compiling" --field branch=main --field commit=9f3a1c2 --ttl 10m
```

It prints `ok` when the control plane took it. Run it again with the same name to update the pane.

| Flag | Meaning |
|---|---|
| `NAME` | The pane's name: 1 to 32 lower-case letters, digits and `-` |
| `--state S` | `idle`, `running`, `needs-you`, `done` or `failed` |
| `--progress P` | A number from 0 to 1 |
| `--title T`, `--message M` | Plain text |
| `--field LABEL=VALUE` | A label and a value; repeat for more |
| `--ttl D` | A duration up to `1h`. With no update for that long, the pane shows as stale. |

---

## Report an HTML pane

The silo must allow `html`. Put the page in a file (`index.html`) or a folder with `index.html` at
its top, then:

```bash
kvo-guest pane html dashboard --bundle ./dashboard --title "Dashboard" --data '{"requests": 120}'
```

The bundle is uploaded once (Kevaso names it by its SHA-256), and the pane then points at it. To push
new numbers to the page without uploading it again, run the same command with new `--data` (JSON,
or `--data -` to read it from standard input). The page defines a function `window.kvoUpdate(data)`,
which the panes app calls with each new value.

| Flag | Meaning |
|---|---|
| `NAME` | As above |
| `--bundle DIR\|FILE` | The pane's page: a folder, a single HTML file, or a ustar archive ([the format](#the-bundle-format)) |
| `--data JSON` or `--data -` | The data pushed to the page, as JSON |
| `--title T`, `--ttl D` | As above |

### The bundle format

A bundle is one of two things:

| Form | What it is |
|---|---|
| **A single HTML page** | Any file that isn't a tar archive. It is the pane's `index.html`, byte for byte |
| **An archive** | A POSIX **ustar** archive with `index.html` at its top, and any files it needs beside it |

`--bundle DIR` makes the archive for you. It packs the folder **the same way every time**: names in
sorted order, no times, no owners, fixed modes. So the same files always give the same digest, and
a pane whose files haven't changed is never uploaded again.

The rules, checked in the silo before anything is sent (and again by the app that opens it):

| Rule | |
|---|---|
| Size as uploaded | At most **512 KiB**. There is no compression: a pane that needs more should load less |
| Files | At most **256**, unpacking to at most **8 MiB** |
| Entries | Regular files and folders only. No links, devices or pipes |
| Names | Relative. No leading `/`, no `..`, no backslash, no control characters, under 256 bytes |
| Duplicates | A file twice is refused |
| Archive extensions | No pax or GNU extension headers: plain ustar |

A folder with a link or a special file in it is refused, not followed. Anything that breaks a rule
is refused with the reason. A bundle uploaded some other way (a raw broker call) isn't checked
until a person opens the pane, and the app then refuses it.

---

## Watch panes with `kvo`

Panes are objects named `SILO.PANE`, in the silo's namespace:

```bash
kvo get panes -n team
```

```text
NAME            SILO     PANE   KIND    STATE    UPDATED  AGE
builder.build   builder  build  status  running  4s ago   2h
builder.dash    builder  dash   html    -        1m ago   2h
```

To follow one silo's panes as they change:

```bash
kvo pane watch builder -n team
```

```text
14:02:11 team/builder [build] running 40% Release build · compiling branch=main commit=9f3a1c2
14:02:41 team/builder [build] running 70% Release build · linking branch=main commit=9f3a1c2
14:03:20 team/builder [dash] html sha256:3b1f7c0a9d… (open it in the panes app)
```

`kvo` shows text only. It removes every control character from what a silo wrote (so a silo can't
move your cursor or rewrite your screen) and puts it on one line. An HTML pane is shown as a
line; opening it is for the panes app.

To save an HTML pane's page to a file, to look at it yourself (the pane is named `SILO.PANE`):

```bash
kvo pane bundle builder.dash -o dash.bundle -n team
```

It's never printed to the terminal, only written to the file.

You need `get`, `list` and `watch` on `panes`. The built-in `viewer`, `editor` and `admin` roles
have them; `admin` can also delete a pane. Nobody can create or change one through the API: only
the silo's own report does.

---

## Limits

The control plane enforces these. A report that breaks one is refused, and `kvo-guest` prints why.

| Limit | Value |
|---|---|
| Text in one status report (title, message, state, labels and values together) | 16 KiB |
| An HTML pane's data | 64 KiB, valid JSON |
| An HTML bundle | 512 KiB |
| Reports per silo | 4 a second sustained, bursts of 8 |
| `--ttl` | Up to 1 hour |
| Panes per silo | `spec.panes.maxPanes` (default 4, at most 16) |
| A silo's stored bundles | The ones its panes use, plus a few newer ones; older ones are dropped |

Text must be UTF-8. When a silo is deleted, its panes and bundles go with it.

---

## Why a pane can't hurt you

The silo is the less trusted side, so the rules all point away from it:

- **One way.** The control plane keeps the latest report and relays it. There is no call that takes
  anything from a pane back to a silo, and no write to a pane except the silo's own report.
- **Who a pane is comes from Kevaso, not the pane.** The silo's name and the pane's name are taken
  from the silo's private channel to its host, not from what it wrote. A client draws them itself,
  outside the pane's own space, so a pane can't pose as another silo.
- **Status panes are only text.** Clients draw them from the fields, with their own layout, with no
  web page, no JavaScript and no markup. Fields that contain HTML or markdown show up as that
  text.
- **The control plane never opens a bundle.** It checks the size and that the bytes match the
  SHA-256 the silo gave, and stores the bundle as an opaque blob. It never unpacks, parses or
  renders it. It's served only as a download (`application/octet-stream`, `attachment`, no
  sniffing, a content security policy that allows nothing), so a browser pointed at it saves a
  file instead of running it. A bug in how pages are read can't be a bug in the control plane,
  because it doesn't read them.
- **The one place a page runs is a client that seals it.** See the panes app, below.

---

## The macOS panes app

The panes app is a small macOS app that shows a person's panes. It lists your clusters, shows each
cluster's panes, and lets you turn any of them into a small window that floats over your work.
Status panes are drawn by the app as plain text. HTML panes run in a sealed view with no network, no
storage, no clipboard, no keyboard and no way to open anything else.

| | |
|---|---|
| **Needs** | macOS 14 or later. No admin rights. Other systems have no client yet. |
| **Install** | Not packaged yet, so there is nothing to install. To try it before it's packaged, see the source's [README](https://github.com/dlebee/kevaso/blob/main/apps/panes/README.md). |

### Sign in

Press **Add Cluster…** and give what you'd give `kvo login`: the server and the CA fingerprint
(`sha256:…`), plus a name and a namespace.

1. The app downloads the CA, and accepts it **only if it matches your fingerprint**.
2. It checks that the server really holds a certificate from that CA.
3. It opens your browser to sign in, with the [invite code]({{ '/access/' | relative_url }}).

It's the same sign-in as `kvo login` (PKCE, a loopback redirect), and uses `kvo`'s public client. Add as
many clusters as you like and switch between them in the sidebar; right-click one to sign in again,
sign out or remove it. Only one cluster's panes float at a time.

| | |
|---|---|
| **Your session** | A refresh token in the macOS login keychain, as `com.kevaso.panes`. It's separate from `kvo`'s (`com.kevaso.kvo`): signing out of one leaves the other. Access tokens stay in memory |
| **Cluster list** | `contexts.json` in Application Support: server, CA and namespace. Never a token |
| **The CA is pinned** | Every request (API, watch, bundle, token, the sign-in pages) trusts **only that cluster's CA**, checks the host name or address, follows no redirect, and fetches no extra certificate data. Without a pinned CA the app refuses to connect: it never uses the system's roots |
| **Sign out** | Tells the control plane to end the session (`POST /oidc/revoke`), then deletes the keychain item. If the cluster can't be reached, you're signed out locally and the session ends when idle |
| **A copy of your token got out** | If the control plane sees an old refresh token used again, it ends the session for everyone. The app deletes its keychain item, closes that cluster's panes and shows that it's signed out |

For CI and scripts, the app also reads these environment variables. When they are set, they win
over the clusters you added:

| Variable | Meaning |
|---|---|
| `KVO_SERVER` | The cluster's address, like `https://203.0.113.10:6443` |
| `KVO_CA_FILE` | The cluster's CA certificate file |
| `KVO_TOKEN` | A [service-account token]({{ '/access/' | relative_url }}#tokens-for-ci) that may read panes and silos in that namespace |
| `KVO_NAMESPACE` | The namespace to show |

### Floating panes

Turn on a pane in the main window, and it becomes a small window that floats above the others, on
every desktop. It never takes the keyboard (a click in a pane leaves your typing where it was), so
it can sit over your work.

- The header drags it; its edges resize it (from 220×140 to 1400×1000); the × turns the pane off.
- Where each window was is remembered per cluster and pane, and it comes back fitted to the screens
  you have (a window on an unplugged display comes back stacked at the bottom right).
- **A pane can't move, resize, focus or close its window.** What it asks for in its own page is
  ignored. A pane that tries to close itself only silences itself.
- **The rim and header are drawn by the app, outside the pane**: the silo's name, namespace and host,
  and marks for *stale*, *needs you* and *untrusted*. A pane can't pose as another silo.
- **Needs you:** when a pane you turned on changes to `needs-you`, its rim glows, and if you ticked
  "Notify me when a pane needs me" (and allowed it in macOS) a notification says "SILO needs you ·
  PANE · NAMESPACE". The text is the app's, never the silo's. It rings at most once a minute for a pane,
  and a pane can't raise one itself. Notifications need the packaged app, which doesn't exist yet; until then
  the app bounces its Dock icon instead.
- **Untrusted silos get a striped rim.** The app reads the label `kevaso.com/trust: untrusted` on
  the Silo, never anything the pane says. If it can't read the silo, the rim is striped too, and
  says "trust unknown": only a silo it read and found unmarked gets a solid rim. The label is not
  reserved yet: today anyone who may update the silo can set it, and nothing in the control plane
  enforces or copies it.

### Not done yet

- **Not packaged.** There is no signed download and no install command yet.
- **A macOS update can change how the sealed view behaves.** The app has a test with hostile panes
  for this; it is run by people building the app, not by you.
- **The untrusted label isn't reserved or copied into the pane,** so a person who may read panes but
  not silos sees every rim as "trust unknown".
- **The streamed browser, Linux, Windows, a web client:** none yet.
