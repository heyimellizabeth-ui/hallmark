# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Read and follow [AGENTS.md](AGENTS.md) first. It holds the contribution rules (one change per pull request, a failing test before a fix, no dependencies, no em dashes or en dashes anywhere, the pull request headings). [CONTRIBUTING.md](CONTRIBUTING.md) explains the same rules at length. This file only adds what those two leave out.

If the task is to build or design a page with Hallmark, and not to change Hallmark itself, follow `skills/hallmark/SKILL.md` instead of this file.

## Commands

There is no install or build step. Everything runs on Node 18 or later with the standard library only.

```bash
# The four checks CI runs (.github/workflows/verify.yml); all must pass
node skills/hallmark/scripts/refcheck.mjs
node skills/hallmark/scripts/theme.mjs validate skills/hallmark/references/themes/*.md
node skills/hallmark/scripts/theme.mjs index && git diff --exit-code -- skills/hallmark/references/themes/index.json
node --test skills/hallmark/scripts/*.test.mjs skills/hallmark/scripts/fixtures/*/*.test.mjs

# One test file, or one test by name
node --test skills/hallmark/scripts/theme.test.mjs
node --test --test-name-pattern "<pattern>" skills/hallmark/scripts/theme.test.mjs

# Lint a page or folder (add --render for the browser tier, --json for diffable output)
node skills/hallmark/scripts/sloplint.mjs <file-or-dir>

# Preview the website at http://localhost:4173
npm run serve
```

Run `sloplint.mjs` or `theme.mjs` with no arguments to print their full usage.

The browser tests (`capture`, `picture`, `sloplint-render`) skip silently unless `CHROME_PATH`, `HALLMARK_PUPPETEER`, `HALLMARK_TEST_CHROME` and `HALLMARK_TEST_PUPPETEER` are set, so a green run without them says nothing about those three scripts. CONTRIBUTING.md shows how to set the four variables.

## Architecture

The repository holds four things with different audiences. Know which one a change lands in before you write it.

- `skills/hallmark/` is the product. The whole folder is copied to every user's machine, and their agent reads it, so every file added here costs every user context. `package.json` ships only this folder.
- `site/` is usehallmark.com, deployed as it is by Vercel with no build. `site/themes/` and `site/made/` are finished pages the skill built; they double as the corpus sloplint is compared against before and after a linter change.
- `eval/` is an internal benchmark harness (briefs, model arms, a vision judge, a gallery report). It is not shipped and CI does not run it. See `eval/README.md`.
- `.claude-plugin/` exposes the same skill as a Claude Code plugin. The version in the `SKILL.md` front matter is the source of truth: `refcheck.mjs` fails when `package.json` or `theme.mjs` disagrees with it, but nothing checks `plugin.json`, so keep that one in step by hand.

### How the skill fits together

`SKILL.md` is the entry point. It describes the build flow and tells the agent which files in `references/` to load at which step, so the references are read on demand and not all at once. The Node scripts in `scripts/` are the tools that flow calls:

- `theme.mjs` owns the theme catalog: `deal` offers three themes for a prompt, `draw` deals the page anatomy from its job, `scaffold` writes the starting files, `photos` picks from `photos.json`, and `validate` and `index` maintain `references/themes/`. A build records its choices in `.hallmark/log.json` in the user's project so later deals avoid repeats.
- `sloplint.mjs` is the static linter over HTML, CSS and component files. `sloplint-render.mjs` is its browser tier, loaded only for `--render`, and checks the page at eight widths through puppeteer-core.
- `capture.mjs` and `picture.mjs` take screenshots of a real product or a built page. `imagegen.mjs` generates images.
- `install.mjs`, `install-hook.mjs` and `lint-hook.mjs` install the skill from a clone and wire an edit-time Floor check into Claude Code.
- `refcheck.mjs` is the consistency check that ties it all together: references that `SKILL.md` names exist, gate tiers agree, fixtures report what they should, versions and links match.

### Gates live in two places

Every lint rule is a numbered gate with a tier: Floor (never ships, cannot be waived), Reflex (a default that a build may overrule with a reason on the record) or Finish (never blocks). The tier table is duplicated on purpose: `references/gates.md` is what agents read, and the `TIERS` map in `sloplint.mjs` is what the linter enforces. `refcheck.mjs` fails when they disagree, and a gate missing from `TIERS` is treated as Floor. A gate change always touches both.

Static gates are tested by fixture pages under `scripts/fixtures/<name>/` with their expected findings in the `expectations` list in `refcheck.mjs`. Render gates are tested by cases in `sloplint-render.test.mjs`.

### Themes

A theme is one Markdown file of at most 8,000 bytes in `references/themes/` describing a material (paper, type, colour, surfaces, components, texture) and never a page plan. `index.json` beside them is generated by `theme.mjs index` and committed; CI fails when it is stale.
