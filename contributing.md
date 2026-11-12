---
title: Contributing
permalink: /contributing/
---

# Contributing

The code lives in [dlebee/kevaso](https://github.com/dlebee/kevaso), under Apache 2.0.
Sign off every commit (`git commit -s`); CI checks the DCO. There is no CLA.

## Build and test

You need Go 1.27.

```bash
make race        # unit tests, every module
make e2e         # the end-to-end loop on a real machine
```

`make e2e` creates a machine (a Lima VM on macOS, the machine itself on Linux), checks it can run
microVMs, runs the test and tears the machine down.

## These docs

These pages are Markdown in [dlebee/kevaso-docs](https://github.com/dlebee/kevaso-docs), served by
GitHub Pages with no build step. Edit a `.md` file and push to `main`.
