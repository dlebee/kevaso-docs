---
title: Contributing
permalink: /contributing/
---

# Contributing

The code lives in [dlebee/kevaso](https://github.com/dlebee/kevaso), under Apache 2.0.
Sign off every commit (`git commit -s`); CI checks the DCO. There is no CLA.

## Build and test

The core repository's `CONTRIBUTING.md` lists what to install first. Then:

```bash
make vet
```

```bash
make race
```

```bash
make e2e
```

`make vet` checks every module, `make race` runs the unit tests with the race detector (what CI
runs), and `make e2e` is the end-to-end loop on a real machine.

`make e2e` creates a machine (a Lima VM on macOS, the machine itself on Linux), checks it can run
microVMs, runs the test and tears the machine down.

## Developer documentation

Docs for people working *on* Kevaso live with the code, in
[`docs/`](https://github.com/dlebee/kevaso/tree/main/docs) of the core repo: design decisions
(`docs/adr/`) and demo walkthroughs (`docs/demo/`). The design
itself and the research behind it are in separate repositories.

## These docs

These pages are for people *using* and *running* Kevaso. They're Markdown in
[dlebee/kevaso-docs](https://github.com/dlebee/kevaso-docs), served by GitHub Pages with no build
step. Edit a `.md` file and push to `main`; add a page to the sidebar in `_config.yml`
(`sections`).

Pages for users of `kvo` live under `use/`, pages for administrators under `admin/`; the layout
says on each page who it is for. A user page never asks the reader to run `kvoadm`: where a step is
an admin's, it says so and links the admin page. A page that moves keeps its old address with
`redirect_from`.
