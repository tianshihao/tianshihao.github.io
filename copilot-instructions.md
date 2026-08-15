# Copilot Instructions

Hugo (v0.128 extended) blog. Site root: `tianshihao.github.io`. The theme is a
git submodule at `themes/hugo-theme-tokiwa` (detached HEAD).

## Before changing the theme — read the design doc

Read `themes/hugo-theme-tokiwa/DESIGN.md` before touching theme layouts, styles,
or scripts. It defines load-bearing invariants (the divider rhythm, the
drop-cap type system, and how the vim highlight aligns to dividers) and the
color token system (§9 — defined once in `static/css/tokens.css`, referenced
as `var(--token)`). Layout or alignment changes that break those invariants are
the most common source of regressions in this repo.

## After changing the theme — update the design doc

Every theme change (layout, style, script, or color) must be reflected back
into `themes/hugo-theme-tokiwa/DESIGN.md`: new invariants, changed behaviors,
and any token/color additions go in the same commit as the code that made them
stale. New colors are defined as tokens in `static/css/tokens.css` (never raw
literals) and documented in DESIGN.md §9. The design doc is the source of truth
— keep it honest, or the next session will rebuild the wrong invariants.

## Repo facts & commands

- Theme is a git submodule: `themes/hugo-theme-tokiwa` (origin
  `git@github.com:tianshihao/hugo-theme-tokiwa.git`). Theme changes commit in
  the submodule first, then the root bumps the pointer (`git add themes/...`).
- Dev server: `hugo server -D --buildFuture --disableFastRender --port 63217
  --bind 127.0.0.1`. `hugo.toml` changes need a FULL restart (that flag only
  hot-reloads content/templates, not config):
  `pkill -9 -f "hugo server"; lsof -ti:63217 | xargs kill -9; nohup hugo server -D --buildFuture --disableFastRender --port 63217 --bind 127.0.0.1 &`
- Fenced code is highlighted **in the browser** by VS Code's TextMate engine
  (see DESIGN.md §10): `static/js/tm-highlight.min.js` + `static/lib/*` (real
  VS Code grammars + theme JSONs). Adding a language = drop its `.tmLanguage`
  into `static/lib/` + one line in `static/lib/grammars.json` — no rebuild.
  Fences are emitted raw by `layouts/_default/_markup/render-codeblock.html`.
- Colors are design tokens: `static/css/tokens.css` (DESIGN.md §9). Never add a
  raw color literal to the theme — define a token and document it.
- Git: manual commands only (no GitLens/GitKraken MCP); conventional commits;
  never commit `.vscode/settings.json`.
- The user's shell is fish: heredocs break there. Wrap complex commands in
  `bash -c '...'` and avoid heredocs (write scripts to /tmp, run them).

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
