# Dependency triage notes (instructor-only)

Private notes on dependency decisions for this repo. Not intended for
students. Kept so the next refresh doesn't re-litigate the same calls.

## Dependency vulnerabilities

Dependabot routinely reports findings on `main`. **None of them affect
students or the taught material.** Triage below.

### Current state (after August 2026 refresh)

| Lockfile | Before | After |
|---|---|---|
| Root `package-lock.json` (Slidev toolchain) | 18 (2 low / 7 mod / 9 high) | **4** (all high, one root cause) |
| Root, 2026-10-04 (`npm audit fix --ignore-scripts`, Slidev 52.19 → 52.20.1) | 15 | **11** (two unpatched upstream chains, see below) |
| `solutions/javascript/my-task-manager` | 0 | 0 |
 
What fixed it (2026-08-19): regenerating the lockfile (`rm -rf node_modules
package-lock.json && npm install --ignore-scripts`) took Slidev 52.12 → 52.19
and cleared most of it; a `package.json` override `"dompurify": "^3.4.14"`
cleared the `monaco-editor`/`mermaid` → `dompurify` chain.
 
As of 2026-10-04 the second chain is `braces <=3.0.3` via
`@slidev/types → vite-plugin-static-copy → chokidar/fast-glob → micromatch → braces`;
3.0.3 is the latest `braces` release, so there is nothing to override yet.

The remaining 4 are all `image-size <=2.0.2` reached via
`@slidev/cli → @slidev/client → pptxgenjs → image-size`. **There is no patched
`image-size` release** (2.0.2 is the latest and is itself flagged), so no
override can clear them — wait for upstream. `npm audit fix --force` would
*downgrade* `@slidev/cli` to 52.6.0; don't.
 
GitHub's Dependabot banner counts the whole repo (Java/Python exercise
manifests included), so its number will be higher than `npm audit`'s.
 
### Re-running the audit next time
 
```bash
# Root (Slidev toolchain - only affects the laptop building the deck)
npm audit fix --ignore-scripts   # --ignore-scripts avoids a Playwright
                                  # post-install that tries to fetch
                                  # Chromium (can fail behind proxies)
 
# Solution lockfile
cd solutions/javascript/my-task-manager
npm audit fix --ignore-scripts
```

### Python / Java exercises

Not covered by `npm audit`. If the Dependabot count is bothering you:

```bash
# Python
cd exercises/python/weather-app
pip-audit -r requirements.txt

# Java
cd exercises/java/bookstore-api
mvn org.owasp:dependency-check-maven:check
```

These are hands-on exercise codebases — students will be modifying them
in class regardless, so a stale dep or two is not a teaching hazard.

## Known "do NOT do in a hurry" items

- **Do not** run `npm audit fix --force` at the root without testing the
  deck rebuild afterwards. It will bump Slidev across a major version.
- **Do not** upgrade Antigravity CLI mid-class. Pin a version on students'
  machines during the prereq step.
- **Do not** commit a slides PDF. CI builds `antigravity-training-slides.pdf`
  on every push to `main` that touches `slides.md` and attaches it to the
  rolling `slides-latest` release
  (`.github/workflows/build-slides-pdf.yml`). Students get the PDF from
  `https://github.com/kousen/antigravity-training/releases/latest/download/antigravity-training-slides.pdf`.
  Re-run manually from the Actions tab (`workflow_dispatch`) if needed.

## Refreshing against a new Antigravity CLI release

Last done: 2026-08-17 against **agy 1.1.13** (materials had been at 1.0.6); re-checked 2026-08-19 on 1.1.15 (no teaching-relevant changes in 1.1.14/15); re-checked 2026-09-02 on **1.1.24** (Gemini 3.8 Flash added, 3.5 Flash gone; `agy mcp` subcommands; workspace hooks still not loaded — see below); re-checked 2026-10-04 on **1.2.16** (workspace `.agents/hooks.json` now loads; `url` accepted alongside `serverUrl`; Claude 5.5 models; `-p` timeout unlimited by default; app 2.17.0).

The "One Brand, Three Products" slide pins app/IDE/CLI version streams
(2.17.x / 2.5.x / 1.2.x as of Oct 2026). Before each delivery check all three:
`agy --version`, and on macOS
`defaults read "/Applications/Antigravity.app/Contents/Info.plist" CFBundleShortVersionString`
(same for "Antigravity IDE.app"). Update the slide if a stream rolled major.

Ground truth, in order of trust:

1. The installed CLI: `agy --help`, `agy changelog`, `agy plugin --help`,
   `agy models` (needs a TTY — from a script use `script -q /dev/null agy models`),
   `agy -p "/settings"` (dumps every settings key + current value, no quota).
2. Docs shipped inside the CLI:
   `~/.gemini/antigravity-cli/builtin/skills/agy-customizations/docs/*.md`
   (skills, plugins, mcp_servers, rules, hooks, json_configs).
3. https://antigravity.google/docs/{settings,mcp,subagents}?tab=cli — the old
   `/docs/cli/...` paths redirect there (checked 2026-10-04); `/docs/cli/hooks`
   is a 404, use the shipped `hooks.md` instead.
4. https://antigravity.google/docs/cli/reference — **partly stale**: still
   listed `/planning` and `/fast` after 1.1.0 removed them. Cross-check.
   Also client-rendered: `curl` gets no command list, read it in a browser.

Quick probe for whether a slash command really exists:
`agy -p "/foo"` — a real interactive-only command errors with
"not available in print mode"; an unknown one gets answered by the model.
(`/export`, `/agent <task>`, `/planning`, `/fast` all failed this test and
were removed from the materials.)

Facts that bit us this round (don't reintroduce):
- Settings live in `~/.gemini/antigravity-cli/settings.json`, flat keys.
  `~/.gemini/settings.json` is the *old Gemini CLI* file.
- Remote MCP servers use `"serverUrl"`. On 1.2.16 `url` also loads and the
  shipped mcp_servers.md calls `serverUrl` legacy; `httpUrl` is not loaded.
  Keep the examples on `serverUrl` until the docs settle.
- Per-project customizations go under `.agents/` (`mcp_config.json`,
  `skills/`, `agents/`), not `.gemini/`.
- Custom commands are skills (`SKILL.md`), not `commands/*.toml`.
- Hooks: a workspace `.agents/hooks.json` LOADS on 1.2.16 (probe from a
  clean scratch dir with only `.agents/hooks.json` + `scripts/`: `/hooks`
  listed both course hooks; log "loaded 3 named hooks from 2 hooks.json
  file(s)"). It was NOT loaded on 1.1.13–1.1.24; slides, lab and
  config-examples were flipped back to `.agents/` on 2026-10-04. Still true:
  a non-object top-level key (e.g. `_comment`) makes the CLI silently drop
  the whole file.
- `${VAR}` is NOT expanded in `mcp_config.json` (verified 1.1.15: the literal
  string was sent as the Context7 header and rejected). Keys go in the global
  file, literally. Context7 replaced Firecrawl in Lab 6 on 2026-08-19.
- Hooks probe, if it regresses again (1.1.24 quirk: a `.agents/hooks.json`
  under an `--add-dir` directory loaded even when the primary workspace's
  did not, so probe the primary workspace). Probe at
  the repo root: `mkdir -p .agents && cp config-examples/hooks.json .agents/
  && cp -r config-examples/scripts .agents/ && agy -p "/hooks"`, then
  `rm -rf .agents`.
- Bare `!` "persistent shell mode" toggle is Gemini CLI, not agy (Ken
  confirmed live, Aug 2026; docs don't mention it). `!command` works;
  `ctrl+b` backgrounds a running one.
- Checkpointing is a Gemini-CLI feature, not agy. agy has `/rewind`, `/diff`,
  `/fork`, diff-review-before-write.

## Exercise repos

- Live-delivery solutions live on per-delivery branches, never `main`:
  `agy_aug2026` (Aug 2026: bookstore-api hardening + my-task-manager build),
  `weather-app-demos`. After a delivery, cherry-pick slide/lab fixes from the
  delivery branch to `main` and leave the solution commits behind.
- `exercises/python/weather-app` on `main` is the **student baseline**.
  The 2026-08-19 restore removed generated reports, caching and
  ARCHITECTURE.md; the June config/error-handling commits (`app/config.py`,
  `app/exceptions.py`, `app/routes/errors.py` — 1c1dbca, 722b088) are still
  on `main` and are part of the baseline. Other live-demo results live on
  branch `weather-app-demos`. Demo on a branch or `git stash` afterwards — if the
  app arrives in class fully tested, Lab 4 has nothing to generate.
- `statusline.py` in weather-app is referenced from the Status Line slide and
  from `~/.gemini/antigravity-cli/settings.json` on Ken's machine.

The `course-refresh-preflight` skill has a config for this repo
(`~/.claude/skills/course-refresh-preflight/configs/antigravity-training.yml`);
run it first — it writes a file:line report without touching the repo.
