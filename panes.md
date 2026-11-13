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

The silo must allow `html`. Put the page in a file (one `index.html`), then:

```bash
kvo-guest pane html dashboard --bundle ./dashboard.html --title "Dashboard" --data '{"requests": 120}'
```

The file is uploaded once (Kevaso names it by its SHA-256), and the pane then points at it. To push
new numbers to the page without uploading it again, run the same command with new `--data` (JSON,
or `--data -` to read it from standard input). The page defines a function `window.kvoUpdate(data)`,
which the panes app calls with each new value.

| Flag | Meaning |
|---|---|
| `NAME` | As above |
| `--bundle FILE` | The pane's page, as one HTML file or an archive the client unpacks |
| `--data JSON` or `--data -` | The data pushed to the page, as JSON |
| `--title T`, `--ttl D` | As above |

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

`apps/panes` in the core repository is a small macOS app that shows a person's panes in one
window: status panes drawn natively, HTML panes in a sealed web view with no network, no storage,
no clipboard, no keyboard and no way to open anything else. It needs macOS 14 or later and Swift 6
(Xcode or the command line tools), and no admin rights.

Build and run it:

```bash
cd apps/panes
```

```bash
swift build
```

```bash
KVO_SERVER=https://203.0.113.10:6443 KVO_CA_FILE=ca.pem KVO_TOKEN=… KVO_NAMESPACE=team swift run KevasoPanes
```

`KVO_TOKEN` is a [service-account token]({{ '/access/' | relative_url }}#tokens-for-ci) that may
read panes and silos in that namespace, and `KVO_CA_FILE` is the cluster's CA certificate. Without
these variables, open the app's Settings (⌘,), fill in the server, CA file, token and namespace
and press Connect. Then turn on the panes you want in the list: only those are shown and opened.

**Not done yet:**

- **Sign-in.** The app doesn't use your `kvo login`. The token comes from the environment or the
  settings, and the settings keep it in the app's preferences as plain text.
- **No packaging.** It runs with `swift run`: no signed or notarised build.
- **The untrusted-silo stripe.** The control plane doesn't say whether a silo is untrusted, so
  every pane's rim is one solid colour.
- **A private WebKit API.** The sealed view depends on some WebKit settings Apple doesn't document,
  so a macOS update can change them. The app has a test suite for exactly this (`swift test`, about
  2½ minutes, on a logged-in desktop). Run it after each macOS update.
- **One leak is unmeasured.** A page asking for a name's DNS address is switched off, but nobody
  has measured that it never goes out.
- **The bundle format.** The app takes a plain `index.html`, or a tar archive (ustar) with an
  `index.html` in it. The control plane doesn't define a format yet; it stores whatever the
  silo sends.
- **Linux, Windows, a web client:** none yet.
