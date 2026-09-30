# Obsidian Mermaid swimlane investigation: learnings, mistakes, decisions

## Goal
Render `swimlane-beta` diagrams in Obsidian through the Mermaid Next plugin
(fork: `KinJLy/obsidian-mermaid-next-plugin`), with lanes drawn as parallel
bands.

## Outcome
Swimlanes now render. Two separate problems were in the way:

1. Obsidian's bundled Mermaid is too old (see "Learnings").
2. With **ELK layout engine** on, the plugin's global `layout: "elk"` overrode
   the swimlane layout, so `swimlane-beta` drew as a plain ELK flowchart with
   subgraph boxes. Fixed in commit `507b55b`.

Commits on `main` (pushed to the fork):

| Commit | What |
| --- | --- |
| `bf4deb2` | Isolate ELK loader registration so a failure no longer discards the CDN Mermaid |
| `507b55b` | Keep `swimlane-beta`'s own layout when ELK is the global default |
| `e19db9c` | Lockfile, the patch file and these notes |

## Learnings

### Mermaid and Obsidian
- Obsidian bundles a fixed Mermaid version that lags upstream. There is no
  setting to bump it, and `window.mermaid` is normally undefined (Obsidian
  loads Mermaid lazily through `loadMermaid()`).
- `swimlane-beta` was added in Mermaid 11.16.0. It is a flowchart-style
  diagram with `defaultLayout: "swimlane"`. It does not give manual node
  positioning.
- For manual placement use `block-beta` (explicit `columns`, `space`, nested
  blocks). If freeform layout is really needed, use a canvas tool (Excalidraw
  or draw.io). Those formats don't diff cleanly.
- Layout precedence in Mermaid (chunk `chunk-SHT3W25Y.mjs`):
  `getUserDefinedConfig().layout ?? defaultLayout ?? cnf.layout`. Config
  passed to `initialize()` counts as user config, so it beats a diagram's
  `defaultLayout`. Frontmatter and directives beat `initialize()` config.
- `"curve": "step"` in a flowchart gives right-angle edges with the default
  dagre layout. It looks like ELK but is not.

### The plugin
- Only ` ```mermaid-next ` blocks go through this plugin. Plain
  ` ```mermaid ` blocks use Obsidian's own Mermaid unless **Replace Obsidian's
  Mermaid** is on.
- CDN mode never auto-fetches. It reads a disk cache filled by the manual
  **Download** button. If the cache is empty it silently falls back to the
  bundled Mermaid (11.15) with no error or warning.
- Changing settings does not always take effect on already-rendered blocks
  or the cached Mermaid instance. Toggling the plugin off and on (or reloading
  the app) was needed.
- `@mermaid-js/layout-elk` is pinned to `^0.2.1` to match the bundled core
  (`^11.15.0`), so it may not suit a newer CDN core.

## Mistakes and dead ends
- **Spent too long on the plugin before reading its source.** Reading
  `src/load-mermaid.ts` earlier would have shown the cache-only CDN design.
- **Named the wrong root cause first.** I diagnosed the "No diagram type
  detected" error as an ELK loader incompatibility silently discarding the CDN
  instance. That is a real code-path risk, and the patch for it is harmless,
  but it was never confirmed: no ELK warning was observed in the console. The
  cause that actually blocked swimlanes was the global ELK layout override.
- **Stated the ELK-overrides-swimlane cause before verifying Mermaid's layout
  precedence.** It turned out to be right, and I checked the source before
  writing the fix, but the first statement was an inference.
- **A shell/Python heredoc corrupted `\b` into a backspace character** when I
  first wrote the swimlane regex. Caught by `cat -A`. Lesson: write regexes
  and escapes through a raw string in a script file, then inspect the result.
- **Misread a diagram as ELK.** The `"curve": "step"` flowchart was dagre.
  Asking for the diagram source resolved it.

## Decisions
- Patch the plugin rather than switch tools, since plain-text Mermaid was the
  requirement.
- Keep manifest `id` (`mermaid-next`) and `version` (`1.2.0`) unchanged. The
  id is stable API, and keeping it lets the patched build replace the installed
  copy and keep its settings. The patched build is marked only through the
  packaged manifest `name` ("Mermaid Next (patched)") and `description`.
- Fix the ELK/swimlane conflict by injecting `layout: swimlane` frontmatter
  (only for `swimlane-beta` sources that lack their own frontmatter) instead of
  dropping the global ELK default. Other diagrams keep ELK.
- Build artifacts stay out of git. The installable package lives outside the
  repo in `C:\Users\KinJLy\Documents\GitHub\mermaid-next-patched\`.
- Committed `package-lock.json` and the applied patch file. The patch file is
  redundant with `bf4deb2` and can be removed.

## Open items
- Diagrams with their own frontmatter do not get `layout: swimlane` injected.
  Add it by hand there.
- The `swimlane-beta` fix has been built and packaged but was not observed
  rendering with ELK turned back on. Test: ELK on, plugin toggled off and on,
  render a swimlane and an ordinary flowchart.
- Consider sending the fix upstream (`dacrystal/obsidian-mermaid-next-plugin`).
- `npm install` reported audit warnings that were not investigated, and esbuild's
  postinstall script is blocked by npm (the build still works).
