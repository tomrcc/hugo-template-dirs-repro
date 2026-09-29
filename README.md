# Hugo editable-regions: templates in Hugo's new layout dirs aren't bundled

Minimal Hugo site for reproducing a Visual Editor failure in
[`CloudCannon/editable-regions`](https://github.com/CloudCannon/editable-regions)
(`v0.0.21`, Hugo `0.164.0`). The module is vendored in `_vendor/`, so the build
doesn't need Go.

Hugo 0.146 renamed some template directories. It still reads the old names, so
this site uses both, side by side on one page:

| Template | Legacy dir (pre-0.146) | New dir (0.146+) |
| --- | --- | --- |
| Partials | `layouts/partials/` | `layouts/_partials/` |
| Shortcodes | `layouts/shortcodes/` | `layouts/_shortcodes/` |
| Render hooks | `layouts/_default/_markup/` | `layouts/_markup/` |

`hugo` builds the site cleanly, using every template in both styles.

## Steps

1. Create a CloudCannon site from this repo, or run `cloudcannon dev public`
   after building.
2. Open the home page in the Visual Editor.

## Expected

All three components render exactly as they do in the built site.

## Actual

| # | Component | Template location | Visual Editor |
| --- | --- | --- | --- |
| 1 | `card-legacy` | `layouts/partials/card-legacy.html` | Renders |
| 2 | `card-new` | `layouts/_partials/card-new.html` | Fails: `No Hugo partial found for component "card-new"` |
| 3 | `markdown` | Heading hook in `layouts/_default/_markup/`, link hook in `layouts/_markup/` | Renders, but the link hook is silently skipped. The link comes out as a plain `<a>` instead of the hooked version. |

The editor's file bundle (`window.cc_hugo_files`) contains only:

```
hugo.toml
layouts/_default/_markup/render-heading.html
layouts/partials/card-legacy.html
layouts/partials/markdown.html
```

The new-dir templates are missing: `layouts/_partials/card-new.html` and
`layouts/_markup/render-link.html`.

## Cause

There are two separate gaps. Fixing only the first isn't enough for partials.

1. **Discovery skips the new dirs.**
   `integrations/hugo/hugo-module/layouts/partials/editable-regions/find-template-files.html`
   walks the project's `layouts/` directory. It only bundles templates from
   directories named `partials` or `shortcodes`, plus `_default/_markup`, so
   `_partials`, `_shortcodes` and `_markup` are skipped.

   This only affects the site's own `layouts/`. Themes and modules are
   discovered through their mounts (`find-dep-template-files.html`), which walk
   the whole layouts tree.

2. **The renderer's component lookup only checks legacy names.** Before
   rendering, the dispatch template in `integrations/hugo/renderer/main.go`
   checks `templates.Exists "partials/<name>"`, then `.html` and `.htm`
   variants. In Hugo 0.164, `templates.Exists` returns `false` for a template
   in `_partials/` unless it's asked for by its new path:

   | Lookup | Template in `layouts/_partials/card-new.html` |
   | --- | --- |
   | `templates.Exists "partials/card-new.html"` | `false` |
   | `templates.Exists "_partials/card-new.html"` | `true` |
   | `partial "card-new"` | renders |

   So a component in `_partials/` is reported as missing even when it is
   bundled. I checked this by forcing the template into the bundle: the
   `_markup` link hook then worked, but `card-new` still failed with
   `No Hugo partial found`.

## Workaround

Use the legacy directory names (`partials/`, `shortcodes/`,
`_default/_markup/`) for anything the editor renders.

## Suggested fix

- Discovery: add `_partials` and `_shortcodes` to the directories the walk
  bundles, and bundle any `_markup` directory. Hugo 0.146+ also allows
  per-section render hooks, such as `layouts/docs/_markup/`.
- Renderer: also check `_partials/<name>` (with and without an extension) in
  the dispatch template's `templates.Exists` lookup.

## Related

Setting `params.editable_regions.template_dirs` bundles the new dirs, but it
also stops `_vendor/modules.txt` and vendored module files from being bundled.
On this site, the editor's Hugo then fails with
`failed to download module github.com/CloudCannon/editable-regions@v0.0.21: binary with name "go" not found in PATH`.
