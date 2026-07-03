# Guinevere fork → upstream plan (for whoever picks up GitLab #48 / #91)

This file is a handoff. It's deliberately scoped separately from Turian's own
in-game GUI work (GitLab issue #47 on `mass4org/mega4/turian`) so that work can
proceed without waiting on this.

## Where things stand

`guinevere` branch is **6 commits ahead of `main`** (all 6 have been submitted
upstream as PRs [#917](https://github.com/david-vanderson/dvui/pull/917),
[#918](https://github.com/david-vanderson/dvui/pull/918), and
[#919](https://github.com/david-vanderson/dvui/pull/919)), fully additive
(`git diff main..guinevere --stat`: +1160/-0 lines, zero lines removed from core
dvui logic):

```
ff920c16 feat: devtools
29f8fca3 feat: multi-frame and new dump APIs
968b1e2c feat: Enriched the dump and Headless CLI — examples/frame-dump.zig
087b8664 feat: machine-readable widget-tree JSON dump #2
3e16ae53 feat: CPU phase timing #3
3ab3f15e feat(window): per-frame render stats (draw calls, triangles, vertices, texture binds/creates)
```

Two new files (`src/Debug.zig` — frame/widget-tree JSON dump; `src/Profiler.zig` —
devtools loop) plus small, commented, tested hooks added to existing files:
`Window.zig` (render-stats counters + CPU phase timing, `RenderStats`/`FrameTiming`
structs, `renderStats()`/`frameTiming()` accessors, a passing test), `WidgetData.zig`
(phase-boundary + debug-capture hooks in `register()`), `Texture.zig` (texture-create
counter), `render.zig` (draw-call/triangle/vertex counters), `dvui.zig` (export
`Profiler`), `build.zig` (register the two new examples).

**As of Turian's #47 epic kickoff, Turian no longer depends on any of this.**
`frameTiming()`'s one consumer (`studio/ProfilerPanel.zig`) is being reimplemented
natively in Turian (`studio/EditorFrameTiming.zig`, bracketing `win.begin()` /
`win.endRendering()` / `win.end()` — all public vanilla dvui API, no patch needed),
and `renderStats()`/`Debug.zig`/`Profiler.zig` were never actually consumed anywhere
in Turian. Turian's `build.zig.zon` will be re-pinned to plain upstream
`github.com/david-vanderson/dvui` (`main`) once that lands. **This removes urgency,
not value** — upstreaming is still worth doing so the work isn't stranded on a
private fork, and so Turian (or anyone else) can pick pieces back up later without
carrying a diverging branch.

## #91 — "reorganize forked DVUI for upstreaming"

Audited: **the fork does not need structural reorg.** It's already cleanly
separated — 2 new files + small additive hooks, nothing entangled with core dvui
logic, nothing that would block being read/reviewed as independent patches. Treat
#91 as satisfied by doing the actual splitting/submission work below, not by
further restructuring.

## #48 — consume/track the patches

Reframe this issue's checklist: the "Turian integration steps" are mostly done or
moot (frameTiming moved to Turian-native code, render-stats/Debug/Profiler were
never wired up). The real remaining scope is the upstream submission itself:

1. **MR 1 — render-stats counters + CPU phase timing** — **submitted**  
   PR [#917](https://github.com/david-vanderson/dvui/pull/917) — 2026-07-03.  
   Rebased fresh off `upstream/main`; two commits (151 lines), no conflicts.

2. **MR 2 — machine-readable frame/widget-tree JSON dump** — **submitted**  
   PR [#918](https://github.com/david-vanderson/dvui/pull/918) — 2026-07-03.  
   Rebased fresh off `upstream/main`; three commits (695 lines). Minor conflict
   resolution needed: the `087b8664` capture hook in `WidgetData.zig`/`Window.zig`
   was adapted to not depend on frame-timing changes from MR 1.

3. **MR 3 — devtools/Profiler loop** — **submitted**  
   PR [#919](https://github.com/david-vanderson/dvui/pull/919) — 2026-07-03.  
   Stacked on MR 1 + MR 2 (the `Profiler` consumes `renderStats()`/`frameTiming()`
   and `Debug.captureScopeBegin/End`). One commit on top; will rebase onto `main`
   once #917 and #918 land.

4. Split cleanly: `mr-1-render-stats`, `mr-2-json-dump`, and `mr-3-devtools`
   branches live in `brmassa/dvui`. `guinevere` stays the staging branch with all
   three.

5. Once (or as) pieces land upstream, note it in this file and in #48 so anyone
   tracking Turian's dependency knows what's safe to pull from plain upstream vs.
   still fork-only.

## Watch list: reusable improvements expected from Turian's GUI epic

*(Added 2026-07-03, after Turian's in-game GUI architecture was finalized —
component-based `.uidoc` documents drawn through a shared tree-walk over dvui,
see `PLAN_GUI_IMPLEMENTATION.md` in the Turian repo.)*

Turian's implementation will exercise dvui in ways Studio never did. Anything
**generic** that surfaces belongs here (fork branch → upstream MR), not in
Turian. Known candidates, roughly in expected order of appearance:

1. **FlexBoxWidget extensions** — `src/widgets/FlexBoxWidget.zig` currently only
   supports row-wrapping + `justify_content` (start/center). Turian v1 uses
   plain `gui.box(dir=h/v)` and doesn't need it, but column direction and
   per-child grow/shrink are the natural first requests when wrap-flow layouts
   arrive. Each is a clean standalone upstream MR.
2. **Image-driven styling gaps** — Turian runs a styling spike early
   (backgrounds/nine-patches/icons/fonts per control, per interaction state,
   resolved from data through a tree-walk rather than hand-written widget
   calls). `Theme.Style` already carries per-state `ninepatch_fill/hover/press`,
   so this may come back clean — but any gap it finds (e.g. per-state icon or
   font handling) is by construction generic dvui territory.
3. **`sdl3gpu-ontop` backend hardening** — Turian's shipped games will be the
   backend's first sustained production consumer (host-owned SDL3-GPU
   device/window, loaned cmd buffer + swapchain per frame). Bug fixes and
   robustness patches found there are prime upstream material.
4. **Stable-ID ergonomics for data-driven trees** — Turian derives widget IDs
   from document-node GUIDs via `Options.id_extra`. If that idiom needs better
   support (e.g. hashing helpers, collision diagnostics), it's generic.
5. **Gamepad/keyboard UI navigation** — post-MVP on Turian's side; upstream
   already has #857 open. Coordinate there rather than fork-tracking a parallel
   implementation.

## Contribution strategy / process rules

- **Separation test**: does the change mention Turian concepts (`.uidoc`,
  assets, GUIDs, Studio)? Then it lives in Turian (`engine/ui` or
  `subsystems/ui_render`). Is it a widget/layout/styling/backend improvement any
  dvui app could want? Then it's fork-track → upstream.
- `guinevere` stays a **thin, always-rebased staging branch**; every upstream
  submission is its own focused branch off current upstream `main` (upstream
  moves ~100 commits/month — rebase fresh immediately before opening each MR).
- Turian pins `build.zig.zon` to the fork branch **only while a patch it needs
  is in flight**, returning to the upstream pin as soon as it lands. Default
  state is the upstream pin.
- New reusable work discovered mid-epic gets its own GitLab issue on the Turian
  tracker (cross-linked to #48) plus an entry in the watch list above — don't
  fold it silently into existing MRs.

## Non-goals here

- No need to hard-fork or diverge further — this file's whole point is un-forking
  once these 3 patch sets are either upstream or deliberately kept as a small,
  documented, always-rebased staging branch.
- No coordination needed on timing with Turian's `build.zig.zon` pin — treat that
  as already resolved/unblocked by the time this is picked up.
