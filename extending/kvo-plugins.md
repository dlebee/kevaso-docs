---
title: kvo plugins
permalink: /extending/kvo-plugins/
---

# kvo plugins

A **kvo plugin** is a program you install next to `kvo` and run as `kvo NAME`. It runs on your own
machine, as you, with your current `kvo` context. It looks the same on macOS, Windows and Linux.

The first one is the [panes client]({{ '/panes/' | relative_url }}), `panes`.

This is a third sort of plugin. [Provider plugins and plugin operators]({{ '/extending/plugins/' | relative_url }})
run on the hosts and are installed with `kvoadm`.

> **No public index exists yet.** `kvo plugin install` reads a signed list of plugins, and Kevaso's
> own list isn't published. Until it is, `kvo plugin install panes` answers "the Kevaso plugin index
> isn't published yet (no key built into this kvo); add an index with kvo plugin index add" and
> installs nothing. You can still use an index you [add](#another-publishers-index)
> yourself, or one you build on your own machine.

---

## Install and run

```bash
kvo plugin search
```

```text
NAME   VERSION  INSTALLED  DESCRIPTION
panes  0.1.0    -          …
```

```bash
kvo plugin install panes
```

```text
installed panes 0.1.0 for darwin/arm64 (sha256 3b1f7c0a9d2e… verified)
run it with: kvo panes
```

```bash
kvo panes
```

| Command | Does |
|---|---|
| `kvo plugin search [TERM]` | Lists the plugins in every index you trust, and whether there is a build for your system |
| `kvo plugin install NAME[@VERSION]` | Installs the newest release with a build for your operating system and CPU. A pre-release only with `@VERSION` |
| `kvo plugin list` | What is installed and from which index |
| `kvo plugin upgrade [NAME]` | Moves one plugin, or all of them, to the newest release |
| `kvo plugin remove NAME` | Removes it |
| `kvo NAME [ARGS…]` | Runs an installed plugin. Its arguments, output and exit code pass through untouched |

A plugin with no build for your system says so, for example "panes has no build for linux/arm64
yet (built for: darwin/arm64)", and exits with an error. There is no emulation fallback.

A plugin never takes the name of a `kvo` command: a built-in command always wins, and `kvo plugin
install` refuses those names.

---

## What a plugin can do

- **It installs into your own folder.** No administrator rights. By default
  `~/Library/Application Support/Kevaso/kvo/plugins` on macOS, `%LOCALAPPDATA%\Kevaso\kvo\plugins` on
  Windows, and `$XDG_DATA_HOME/kevaso/kvo/plugins` (or `~/.local/share/kevaso/kvo/plugins`) on Linux.
  `KVO_PLUGINS_DIR` replaces it. One version of a plugin is installed at a time.
- **Each one is checked before it is installed.** The index is signed, and `kvo` checks the
  signature before reading it. Every download is checked against the SHA-256 in the signed index
  before anything is unpacked. An index older than one `kvo` has seen is refused.
- **It gets your context, never your sign-in.** `kvo` starts the plugin with the cluster's address,
  the cluster's CA and your default namespace in `KVO_PLUGIN_SERVER`, `KVO_PLUGIN_CA_FILE` and
  `KVO_PLUGIN_NAMESPACE` (and `KVO_PLUGIN_CONTEXT`, `KVO_PLUGIN_NAME`, `KVO_PLUGIN_VERSION`,
  `KVO_PLUGIN_DIR`, `KVO_PLUGIN_KVO`). When it needs a token it runs `kvo plugin token`, which
  prints a short-lived one. Your refresh token stays with `kvo`.
- **It has your permissions and no more.** A context that signs in with a client certificate (the
  one `kvoadm init` writes) is refused: `kvo` never hands a plugin a key.

---

## Where the index comes from

An index is one signed file listing plugins, their versions and a build per operating system and CPU.

| Source | How |
|---|---|
| Kevaso's own | Built into `kvo`. **Not published yet**, so it has no key and `kvo` says so |
| Another publisher's | You add it on purpose, with its key |
| One of your own | `KVO_PLUGIN_INDEX` (a URL) and `KVO_PLUGIN_INDEX_KEY` replace the built-in pair, for a local or staging index |

### Another publisher's index

```bash
kvo plugin index add https://plugins.example.com/index.json --key ed25519:…
```

```text
added index acme (signature verified); its plugins are acme/NAME
```

The plugins of another index always carry its name, so one can never pass for Kevaso's:

```bash
kvo plugin install acme/foo
```

```bash
kvo acme/foo
```

| Command | Does |
|---|---|
| `kvo plugin index add URL --key ed25519:… [--name NAME]` | Trusts an index, after checking its signature with the key you gave. The name is the one the index suggests, or `--name`. It is never `kevaso` |
| `kvo plugin index list` | Name, address and key of each index |
| `kvo plugin index remove NAME` | Stops trusting it. Refused while a plugin from it is installed |

Indexes are fetched over HTTPS (plain HTTP only to your own machine).

---

## Writing one

A plugin is any program that reads the variables above and calls the API with the token it is
given. Core's repository has a small example, `examples/kvo-plugin`, with a README that states the
contract and how to build and sign an index.

---

## Not done yet

- **The project's index and its key are not published.** Until they are, nothing installs from
  `kvo plugin install panes` without an index you add.
- **Windows and Linux builds of `panes`.** The panes client exists for macOS only.
- **No key rotation or freshness limit for an index yet:** an index stays valid until a newer one
  replaces it.
- **Artifacts of the macOS client are not notarised.**
