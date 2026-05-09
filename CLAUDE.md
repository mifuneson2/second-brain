# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is the **source repository for the `second-brain` Agent Skills package**, distributed to end users via `npx skills add NicholasSpisak/second-brain`. It is **not** a Second Brain vault — it ships the four skills (`/second-brain`, `/second-brain-ingest`, `/second-brain-query`, `/second-brain-lint`) that users install into their AI agent (Claude Code, Codex, Cursor, Gemini CLI, etc.) to scaffold and maintain an Obsidian-based LLM Wiki.

When working here you are editing **the skills themselves**, not running them.

## Commands

- `bash tests/test_onboarding.sh` — runs the only test in the repo. Exercises `skills/second-brain/scripts/onboarding.sh` end-to-end against a temp dir: directory scaffold, `wiki/index.md` and `wiki/log.md` content, idempotency (re-running must not overwrite), valid JSON on stdout. There is no test runner — it's a pure bash script with `assert_*` helpers and a `PASS/FAIL` tally.
- `bash skills/second-brain/scripts/onboarding.sh <vault-path>` — run the scaffolder directly. Progress goes to stderr; a JSON summary goes to stdout. Idempotent.

There is no build step, no package manager, and no lint command. The repo is markdown + bash.

## Architecture

### Single source of truth: `skills/second-brain/references/wiki-schema.md`

This file holds the canonical wiki rules (architecture, page format, operations, naming, image handling, the 10 numbered rules). The four agent-config templates in `skills/second-brain/references/agent-configs/` (`claude-code.md`, `codex.md`, `cursor.md`, `gemini.md`) all interpolate it via the `{{WIKI_SCHEMA}}` placeholder when the onboarding wizard generates an end user's agent config.

**If you change wiki rules, change them here only.** Do not duplicate rules into individual `SKILL.md` files or agent-config templates — those should reference or be filled from the schema.

### Skill layout

Each skill is a directory under `skills/` containing a `SKILL.md` with YAML frontmatter (`name`, `description`, `allowed-tools`). The onboarding skill additionally ships:

- `scripts/onboarding.sh` — pure bash, no dependencies beyond `python3` for JSON pretty-printing in the tool-detection loop
- `references/` — schema + agent-config templates + tooling docs, loaded by the skill at runtime

The other three skills (`second-brain-ingest`, `second-brain-query`, `second-brain-lint`) are pure prompts — `SKILL.md` only.

### What an installed vault looks like (for context)

End users get a vault with three top-level dirs: `Clippings/` (immutable inbox, named to match the Obsidian Web Clipper default), `wiki/` (LLM workspace with `sources/`, `entities/`, `concepts/`, `synthesis/`, plus `index.md` and `log.md`), and `output/`. The agent config (`CLAUDE.md`/`AGENTS.md`/etc.) at the vault root is generated from the templates in this repo. Keep this mental model when editing skill instructions — the skills run *in the user's vault*, not here.

## Conventions that matter when editing skills

These are non-obvious rules embedded in `second-brain-ingest/SKILL.md` that exist because they prevented real bugs. Preserve them when editing:

- **Straight apostrophes only** (`'`, ASCII 39) in titles, `aliases:`, and `[[wikilinks]]`. Curly apostrophes (`'`, U+2019) cause Obsidian to spawn empty duplicate pages because wikilinks won't resolve.
- **Source pages need an `aliases:` frontmatter field** matching the H1 exactly. Source filenames are kebab-case but their H1s and inbound wikilinks are Title Case — without aliases, Obsidian can't resolve the link.
- **YAML aliases containing double quotes** must be wrapped in single quotes (`aliases: ['Title with "Quoted" Words']`) or YAML parsing breaks.
- **Wikilink style differs by page type**: source pages → link by H1 title (not slug); entity/concept pages → link by Title Case name that matches the filename. Both `SKILL.md` and the schema document this; keep them in sync.

## Distribution model

The skills here are consumed via the [skills.sh](https://agentskills.io) ecosystem — `npx skills add` copies them into agent-specific directories (`.claude/`, `.codex/`, `.cursor/`, `.gemini/`, etc.). The long list of agent dirs in `.gitignore` exists because users running `npx skills add` *inside this repo* would otherwise commit install artifacts. Don't remove those entries.
