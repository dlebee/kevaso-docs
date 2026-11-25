# kevaso-docs

Documentation for [Kevaso](https://github.com/dlebee/kevaso), live at
https://dlebee.github.io/kevaso-docs/.

These are the docs for people *using* Kevaso (`use/`, the `kvo` side) and *running* it (`admin/`,
the `kvoadm` side); docs for people working on Kevaso live in the core repo's `docs/`.

Plain Markdown served by GitHub Pages (built-in Jekyll, no theme, no build step). Add a page by
creating a `.md` file with a `title` and `permalink`, then add it to a section of the sidebar
(`parts`, then `sections`, in `_config.yml`; each part is a collapsible menu). A page that moves keeps its old address with `redirect_from`
(the `jekyll-redirect-from` plugin GitHub Pages ships). Link between pages with `{{ '/path/' | relative_url }}`; a
` ```mermaid ` block becomes a diagram. Wrap literal `{{ … }}` in `{% raw %}…{% endraw %}`, or Jekyll
empties it.
