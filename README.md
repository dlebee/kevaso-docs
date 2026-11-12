# kevaso-docs

Documentation for [Kevaso](https://github.com/dlebee/kevaso), live at
https://dlebee.github.io/kevaso-docs/.

These are the docs for people *using* Kevaso; docs for people working on Kevaso live in the core
repo's `docs/`.

Plain Markdown served by GitHub Pages (built-in Jekyll, no theme, no build step). Add a page by
creating a `.md` file with a `title` and `permalink`, then add it to a section of the sidebar
(`sections` in `_config.yml`). Link between pages with `{{ '/path/' | relative_url }}`; a
` ```mermaid ` block becomes a diagram.
