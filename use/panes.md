---
title: Panes
permalink: /use/panes/
redirect_from: /panes/
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

## How a silo reports

**Nothing of Kevaso's is installed in the silo.** The silo serves its panes over plain HTTP, on a
port it chooses, and its host fetches them. Say where in the silo's spec:

```yaml
spec:
  panes:
    http: { port: 7070, path: /panes }
    every: 5s
    allow: [status, html]
    maxPanes: 4
```

| Field | Meaning | Default |
|---|---|---|
| `http.port` | The port the silo answers on. Without `http`, panes are off and nothing is fetched | |
| `http.path` | The path of the list | `/panes` |
| `every` | How often the host asks | `5s`. At least `1s`, at most `5m` |
| `allow` | Which kinds the silo may report | `status` only |
| `maxPanes` | How many panes it may have | 4, at most 16. `0` is the same as leaving it out |

The host asks from the host itself, at the silo's own private address, with nothing from clients in
the request. It follows no redirect. The silo implements two `GET`s, with any web server:

| Request | Answer |
|---|---|
| `GET /panes` | `application/json`: `{"panes": [REPORT, …]}` |
| `GET /panes/bundles/sha256:HEX` | An `html` pane's [bundle](#the-bundle-format), asked once per digest |

A report is a JSON object. Fields the silo doesn't need can be left out; fields Kevaso doesn't know
are refused. `silo` and `siloUID` are ignored: Kevaso fills them in.

| Field | Meaning |
|---|---|
| `pane` | The pane's name. The object is `SILO.PANE` |
| `kind` | `status` or `html` |
| `title`, `message` | Plain text |
| `state` | `idle`, `running`, `needs-you`, `done` or `failed` |
| `progress` | A number from 0 to 1 |
| `fields` | A list of `{"label": …, "value": …}` |
| `bundle` | `html` only: the bundle's digest, `sha256:` and 64 hex digits |
| `data` | `html` only: JSON pushed to the page |
| `ttl` | Past it without an update the pane is stale. Default: three times `every`, at least 10 seconds. At most 1 hour |

For example, with a web server answering `GET /panes`:

```json
{"panes": [{"pane": "build", "kind": "status", "title": "Release build", "state": "running",
            "progress": 0.4, "message": "compiling", "fields": [{"label": "branch", "value": "main"}]}]}
```

If the silo answers wrongly, too much or too slowly, its host stops asking for a minute, then
longer (up to 15 minutes), and starts again after an answer that's fine. What counts as wrong:
a redirect, a status other than 200, a content type other than JSON, more than `maxPanes` panes, the
same pane named twice, a kind not in `allow`, a report over the [limits](#limits), a bundle whose
digest doesn't match.
Meanwhile the panes go stale. The silo's `Panes` condition says why: `CutOff`, or `Unreachable` while
nothing answers (which isn't held against it).

---

### The bundle format

A bundle is one of two things:

| Form | What it is |
|---|---|
| **A single HTML page** | Any file that isn't a tar archive. It is the pane's `index.html`, byte for byte |
| **An archive** | A POSIX **ustar** archive with `index.html` at its top, and any files it needs beside it |

The silo builds the bundle and serves it. To keep a pane whose files haven't changed at the same
digest, pack the folder **the same way every time**: names in sorted order, no times, no owners,
modes `0644` and `0755`. Kevaso ships no packer.

The rules, checked by the host when it fetches the bundle (and again by the app that opens it):

| Rule | |
|---|---|
| Size as uploaded | At most **512 KiB**. There is no compression: a pane that needs more should load less |
| Files | At most **256**, unpacking to at most **8 MiB** |
| Entries | Regular files and folders only. No links, devices or pipes |
| Names | Relative. No leading `/`, no `..`, no backslash, no control characters, under 256 bytes |
| Duplicates | A file twice is refused |
| Archive extensions | No pax or GNU extension headers: plain ustar |

A folder with a link or a special file in it is refused, not followed. The host and the control plane
only check a bundle's size and that its SHA-256 is its name; the app that opens it refuses one
outside the format.

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

The control plane enforces these. A report that breaks one is refused. If a silo keeps breaking them, its host stops asking for a
minute, then longer, and the silo's `Panes` condition says why.

| Limit | Value |
|---|---|
| Text in one status report (title, message, state, labels and values together) | 16 KiB |
| An HTML pane's data | 64 KiB, valid JSON |
| An HTML bundle | 512 KiB |
| Reports per silo | 4 a second sustained, bursts of 8 |
| The list the host reads | At most `maxPanes` × 81 KiB, within 2 seconds. A bundle: 512 KiB, within 5 seconds |
| A report's `ttl` | Up to 1 hour |
| Panes per silo | `spec.panes.maxPanes` (default 4, at most 16). `0` is the same as leaving it out |
| A silo's stored bundles | The ones its panes use, plus a few newer ones; older ones are dropped |

Text must be UTF-8. When a silo is deleted, its panes and bundles go with it.

---

## Why a pane can't hurt you

The silo is the less trusted side, so the rules all point away from it:

- **One way.** The control plane keeps the latest report and relays it. There is no call that takes
  anything from a pane back to a silo, and no write to a pane except the silo's own report.
- **Who a pane is comes from Kevaso, not the pane.** The silo's name is the silo the host fetched from, and the
  control plane fills it in. A `silo` field in a report is ignored. A client draws them itself,
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
| **Install** | `kvo plugin install panes`, then `kvo panes`. The index it installs from [isn't published yet](#what-runs-today), so this doesn't work today. |

### What runs today

| You want | Today |
|---|---|
| See panes as text | `kvo get panes` and `kvo pane watch SILO`. They work now |
| Save an `html` pane's page | `kvo pane bundle SILO.PANE -o FILE`. Works now |
| The panes app, installed with `kvo plugin install panes` | **Not yet.** `kvo` answers that the plugin index isn't published, because no index exists. See [kvo plugins]({{ '/use/kvo-plugins/' | relative_url }}) |
| The panes app, any other way | Build it from a checkout of the source and run it. The app's [README](https://github.com/dlebee/kevaso/blob/main/apps/panes/README.md) says how, and how to make a local plugin index so that `kvo plugin install panes` works on your own machine |

Once the app is installed through `kvo`, `kvo panes` starts it in your current `kvo` context: the
server, the CA to trust and the namespace come from `kvo`, and it asks `kvo` for short-lived tokens
when it needs one. It signs nothing in and keeps no token of its own. A context with a client
certificate (the one `kvoadm init` writes) can't be used this way: sign in with `kvo login` first.

### Sign in (when started by hand)

Started on its own, rather than through `kvo panes`, the app signs in itself. Press **Add Cluster…** and give what you'd give `kvo login`: the server and the CA fingerprint
(`sha256:…`), plus a name and a namespace.

1. The app downloads the CA, and accepts it **only if it matches your fingerprint**.
2. It checks that the server really holds a certificate from that CA.
3. It opens your browser to sign in, with the [invite code]({{ '/admin/access/' | relative_url }}).

It's the same sign-in as `kvo login` (PKCE, a loopback redirect), and uses `kvo`'s public client. Add as
many clusters as you like and switch between them in the sidebar; right-click one to sign in again,
sign out or remove it. Only one cluster's panes float at a time.

| | |
|---|---|
| **Your session** | A refresh token in the macOS login keychain, as `com.kevaso.panes`. It's separate from `kvo`'s (`com.kevaso.kvo`): signing out of one leaves the other. Access tokens stay in memory |
| **Cluster list** | `contexts.json` in Application Support: server, CA and namespace. Never a token |
| **The CA is pinned** | Every request (API, watch, bundle, token, the sign-in pages) trusts **only that cluster's CA**, checks the host name or address, follows no redirect, and fetches no extra certificate data. Without a pinned CA the app refuses to connect: it never uses the system's roots |
| **Sign out** | Tells the control plane to end the session (`POST /oidc/revoke`), then deletes the keychain item. If the cluster can't be reached, you're signed out locally and the session ends when idle |
| **A copy of your token got out** | An old refresh token used again within 60 seconds is a race: the session stays (`refresh raced with another; try again`). After 60 seconds the control plane ends the session for everyone (`refresh token was already used; session revoked`). The app deletes its keychain item, closes that cluster's panes and shows that it's signed out |

For CI and scripts, the app also reads these environment variables. When they are set, they win
over the clusters you added:

| Variable | Meaning |
|---|---|
| `KVO_SERVER` | The cluster's address, like `https://203.0.113.10:6443` |
| `KVO_CA_FILE` | The cluster's CA certificate file |
| `KVO_TOKEN` | A [service-account token]({{ '/admin/access/' | relative_url }}#tokens-for-ci) that may read panes and silos in that namespace |
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
  and a pane can't raise one itself. Notifications need the app to run as an app bundle, which the plugin build is; run any
  other way, the app bounces its Dock icon instead.
- **Untrusted silos get a striped rim.** The app reads the label `kevaso.com/trust: untrusted` on
  the Silo, never anything the pane says. If it can't read the silo, the rim is striped too, and
  says "trust unknown": only a silo it read and found unmarked gets a solid rim. The label is not
  reserved yet: today anyone who may update the silo can set it, and nothing in the control plane
  enforces or copies it.

### Not done yet

- **Not published.** There is no plugin index, signed or not, so `kvo plugin install panes` has
  nothing to install. The macOS build is only ad-hoc signed and not notarised, so a release also
  needs a Developer ID signature.
- **A macOS update can change how the sealed view behaves.**
- **The untrusted label isn't reserved or copied into the pane,** so a person who may read panes but
  not silos sees every rim as "trust unknown".
- **The streamed browser, Linux, Windows, a web client:** none yet.
