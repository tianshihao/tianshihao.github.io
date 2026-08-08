# Copilot Instructions

Hugo (v0.128 extended) blog. Site root: `tianshihao.github.io`. The theme is a
git submodule at `themes/hugo-theme-tokiwa` (detached HEAD).

## Before changing the theme — read the design doc

Read `themes/hugo-theme-tokiwa/DESIGN.md` before touching theme layouts, styles,
or scripts. It defines load-bearing invariants (the divider rhythm, the
drop-cap type system, and how the vim highlight aligns to dividers). Layout or
alignment changes that break those invariants are the most common source of
regressions in this repo.

## Environment

- Tailwind is PREBUILT at `static/dist/app.css`; new utility classes are
  ignored. Use inline styles or a `<style>` block in `baseof.html`.
- Dev server:
  `hugo server -D --buildFuture --disableFastRender --port 63217 --bind 127.0.0.1`
- Long-running dev servers can serve stale content — restart and clear
  `resources/_gen` if behavior seems stale.
- Pushing the theme submodule requires `git branch -f master HEAD` first, then
  push origin master.
- The VS Code built-in browser caches inline scripts; hard-refresh
  (Cmd+Shift+R) after script changes.

## Style guidance

- Comments should explain design intent ("why", "what must stay true",
  "non-goals"), not just mechanics. The design doc is the source of truth for
  intent; local comments should reference it where relevant.
