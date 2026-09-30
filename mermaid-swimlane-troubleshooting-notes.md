# Obsidian Mermaid Swimlane Investigation — Notes

## Original goal
Render swimlane diagrams in Obsidian, with nodes arranged in a specific,
predictable layout (not wherever an auto-layout algorithm decides to put them).

## Key learnings

1. **Obsidian bundles its own fixed Mermaid version.** It only updates when
   Obsidian itself ships a new release, and typically lags upstream Mermaid
   by months. There's no built-in setting to bump it.

2. **`swimlane-beta` is a real, new Mermaid diagram type**, added in Mermaid
   **v11.16.0**. It uses `subgraph`/`end` for lanes — i.e. it's built on the
   same flowchart layout engine (dagre/ELK) as regular flowcharts, just
   constrained to render lanes as parallel bands. It does **not** give manual
   node positioning; it inherits the same "layout engine decides" behavior.

3. **Core distinction that mattered most:** Mermaid diagram types split into
   - *auto-layout* types (flowchart, `swimlane-beta`, sequence, class, etc.) —
     you describe relationships, an algorithm decides positions.
   - *manual-layout* types — a small set where Mermaid hands positioning back
     to you.
   The one that actually solves "I need nodes to stay where I put them" is
   **`block-beta` (block diagrams)**: explicit `columns N` grids, `id:N` for
   spanning, `space` for gaps, and nested `block:groupId ... end` to simulate
   lanes. Added to Mermaid in early 2024, so it's very likely already
   supported by Obsidian's stock bundled Mermaid — no plugin needed.

4. **If pixel-perfect / freeform manual layout is the real requirement**
   (not just a grid), no Mermaid diagram type will fully satisfy that — it's
   always some layout engine underneath. At that point a canvas tool is the
   right category of tool, not a text-to-diagram compiler:
   - **Excalidraw** (Obsidian plugin) — freeform, drag-and-drop, well-maintained.
   - **draw.io / diagrams.net** (Obsidian plugin) — has a real swimlane/pool
     shape with lane snapping, also well-maintained.
   - Trade-off vs. Mermaid: both store diagrams as JSON/XML, not readable
     plain text, so they don't diff cleanly in version control.

## Mistakes / dead ends

- **Chased a low-quality community plugin (Mermaid Next) for too long**
  before checking its source. It has a low community trust score, a single
  maintainer, and was already known to have at least one settings-related
  bug (a "Replace Obsidian's Mermaid" toggle that doesn't take effect until
  re-flipped after launch — not actually our issue, but a sign of fragility).
- **Assumed "Source: CDN" + "Version: latest" meant an automatic fetch.**
  In this plugin's design, CDN mode never fetches on its own — it only ever
  reads from a **manually populated disk cache**. If that cache is empty,
  it silently falls back to the old bundled Mermaid with **no error, no
  console warning, nothing** — the failure mode is indistinguishable from
  "the feature doesn't exist" unless you read the source.
- **Even after caching correctly, the bug persisted** — traced to a second,
  more subtle issue (see Root Cause below).

## Root cause (confirmed in code)

Two compounding issues in `src/load-mermaid.ts`, found by cloning the user's
fork (`https://github.com/KinJLy/obsidian-mermaid-next-plugin`):

1. **CDN mode requires an explicit manual "Download" click** in plugin
   settings to populate a disk cache. Setting Source/Version alone does
   nothing at render time.
2. **`mermaid.registerLayoutLoaders(elkLayoutLoaders)` was called
   unconditionally, inside the same `try/catch` as the CDN module import.**
   The plugin pins `@mermaid-js/layout-elk@^0.2.1` to match the *bundled*
   Mermaid core (`^11.15.0`). When a newer CDN-fetched Mermaid core (e.g.
   "latest", likely 12.x) is loaded instead, that pinned ELK loader package
   may not be API-compatible with it. If `registerLayoutLoaders()` throws,
   the whole CDN-loaded instance was discarded and **silently replaced with
   the old bundled 11.15 core** — which is one minor version short of
   `swimlane-beta` (added in 11.16.0). This reproduces the exact
   `"No diagram type detected matching given configuration"` error even with
   a correctly cached CDN download.

## Decision / fix applied

Patched `getMermaid()` in `load-mermaid.ts` (fork: `KinJLy/obsidian-mermaid-next-plugin`,
commit `55044d0`) to:
- Separate ELK layout-loader registration into its own `try/catch`, so a
  failure there degrades to "no ELK" instead of discarding the whole
  CDN-loaded Mermaid instance.
- Only attempt ELK registration when the "ELK layout engine" setting is
  actually enabled (previously it ran regardless of that toggle).
- Log a distinct, specific console warning when ELK registration fails, so
  future failures are diagnosable instead of silent.

Delivered as a patch file (`0001-fix-elk-layout-loader-fallback.patch`) for
the user to apply and push themselves, since Claude has no push credentials
for the user's GitHub repo and does not accept tokens to act as one — this
is a firm boundary, not a configurable preference.

## Open item / next step

Confirm via DevTools console on next render whether the ELK-incompatibility
warning appears. If it does, the patch should resolve it. If a *different*
warning appears (e.g. the blob-URL `import()` itself failing), that's a
separate root cause still to chase.

## Recommended path forward (independent of the plugin bug)

Given the actual requirement — clean, predictable, non-jumping node
placement — the most efficient routes, in order of effort:

1. Try `block-beta` in a **plain `mermaid` code block first** (no plugin) —
   likely already supported by stock Obsidian.
2. If true swimlane visuals (labeled bands, not simulated via grouping) are
   required, use **draw.io** for its native pool/lane shape with manual
   positioning.
3. Keep chasing `swimlane-beta` via Mermaid Next only if staying in
   plain-text/versionable Mermaid syntax is a hard requirement — otherwise
   it's the highest-effort, least certain option of the three.
