---
name: second-brain-ingest
description: >
  Process raw source documents into wiki pages. Use when the user adds
  files to Clippings/ and wants them ingested, says "process this source",
  "ingest this article", "I added something to Clippings/", or wants to
  incorporate new material into their knowledge base.
allowed-tools: Bash Read Write Edit Glob Grep
---

# Second Brain — Ingest

Process raw source documents into structured, interlinked wiki pages.

## Identify Sources to Process

Determine which files need ingestion:

1. If the user specifies a file or files, use those
2. If the user says "process new sources" or similar, detect unprocessed files:
   - List all files in `Clippings/` (excluding `Clippings/assets/`)
   - Read `wiki/log.md` and extract all previously ingested source filenames from `ingest` entries
   - Any file in `Clippings/` not listed in the log is unprocessed
3. If no unprocessed files are found, tell the user

## Process Each Source

For each source file, follow this workflow:

### 0. Identify the canonical title

Before reading the body, capture the canonical title that will become the H1, the `aliases:` value, and the text inside every `[[wikilink]]` pointing to this source. Wikilink resolution depends on this string staying consistent across every page in this ingest, so lock it in early.

1. Open the source file. Find the first H1 (a line starting with `# `). That H1 is the canonical title — the Obsidian Web Clipper preserves the article's real title there.
2. Apply Character Normalization (see the Character Normalization section of your agent config (CLAUDE.md / AGENTS.md / equivalent at the vault root)): substitute curly quotes (`'` `"` `"`) to straight (`'` `"`); preserve em-dashes `—` and ampersands `&` verbatim; drop `/`, `|`, `[`, `]`, `#` if present.
3. If the file has no H1, derive a title by reverse-slugifying the filename (replace `_` and `-` with spaces, drop the `.md` extension, apply title case) and ask the user to confirm before proceeding.
4. Hold this normalized title in working memory. Steps 3, 5, and 9 must use it byte-for-byte — every H1, alias, and wikilink referencing this source must match exactly.

### 1. Read the source completely

Read the entire file. If the file contains image references, note them — read the images separately if they contain important information.

### 2. Discuss key takeaways with the user

Before writing anything, share the 3-5 most important takeaways from the source. Ask the user if they want to emphasize any particular aspects or skip any topics. Wait for confirmation before proceeding.

### 3. Create source summary page

Create a new file in `wiki/sources/` named after the source (slugified). The filename must be kebab-case (e.g., `ben-fellows-how-to-build-pipelines.md`). Include:

    ---
    aliases: ["Exact Source Title Here"]
    tags: [relevant, tags]
    sources: [original-filename.md]
    created: YYYY-MM-DD
    updated: YYYY-MM-DD
    ---

    # Exact Source Title Here

    **Source:** original-filename.md
    **Date ingested:** YYYY-MM-DD
    **Type:** article | paper | transcript | notes | etc.

**`aliases:` must equal the H1 byte-for-byte.** This is what makes `[[Source Title]]` wikilinks resolve to the kebab-slug file. Without an exact-match alias, Obsidian cannot find the file and creates an empty duplicate at the vault root. Apply Character Normalization (see your agent config's Character Normalization section) to both the H1 and the alias — straight quotes, preserved em-dashes/ampersands, dropped `/ | [ ] #`.

**YAML quoting when the title contains `"`**: wrap the alias value in single quotes so YAML doesn't fail to parse:
- ✅ Correct: `aliases: ['Dario Amodei — "We Are Near The End Of The Exponential"']`
- ❌ Wrong: `aliases: ["Dario Amodei — "We Are Near The End Of The Exponential""]` (breaks YAML)

    ## Summary

    Structured summary of the source content.

    ## Key Claims

    - Claim 1
    - Claim 2
    - ...

    ## Entities Mentioned

    - [[Entity Name]] — brief context
    - ...

    ## Concepts Covered

    - [[Concept Name]] — brief context
    - ...

### 4. Update entity and concept pages

For each entity (person, organization, product, tool) and concept (idea, framework, theory, pattern) mentioned in the source:

**Before creating any new page, check for name variants.** Search existing entity, concept, and synthesis pages for a similar name — case-insensitive substring match, common alternate spellings (Marc/Mark, Andrew/Andy), expanded vs. abbreviated forms (`OpenAI` vs `Open AI`), compound vs. split (`Google DeepMind` vs `Google` + `DeepMind`). If a near-match exists, ask the user whether to merge into the existing page rather than create a duplicate. Treat this as the default; only proceed to create when the user confirms the new page is distinct.

**If a wiki page already exists:**
- Read the existing page
- Add new information from this source
- Add the source to the `sources:` frontmatter list
- Update the `updated:` date
- Note any contradictions with existing content, citing both sources

**If no wiki page exists:**
- Create a new page in the appropriate subdirectory:
  - `wiki/entities/` for people, organizations, products, tools
  - `wiki/concepts/` for ideas, frameworks, theories, patterns
- Include YAML frontmatter with tags, sources, created, and updated fields
- Write a focused summary based on what this source says about the topic

### 5. Add wikilinks

Ensure all related pages link to each other using `[[wikilink]]` syntax. Every mention of an entity or concept that has its own page should be linked.

### 6. Update wiki/index.md

For each new page created, add an entry under the appropriate category header:

    - [[Page Name]] — one-line summary (under 120 characters)

### 7. Update wiki/log.md

Append:

    ## [YYYY-MM-DD] ingest | Source Title
    Processed source-filename.md. Created N new pages, updated M existing pages.
    New entities: [[Entity1]], [[Entity2]]. New concepts: [[Concept1]].

### 8. Report results

Tell the user what was done:
- Pages created (with links)
- Pages updated (with what changed)
- New entities and concepts identified
- Any contradictions found with existing content

### 9. Validate wikilinks before declaring done

Before finalizing the ingest, verify every `[[wikilink]]` written or touched in this run resolves. A failed validation means broken links land in the wiki and Obsidian creates ghost pages on click.

1. **Collect** every `[[wikilink]]` text from each page created or modified in this ingest (the source page, every entity/concept/synthesis page touched, and the index entries you added). Use:
   ```bash
   grep -roh '\[\[[^]]*\]\]' wiki/ | sed 's/\[\[\(.*\)\]\]/\1/' | sort -u
   ```
   For piped wikilinks (`[[slug|Display Text]]`), the link target is the part before the `|`.

2. **Resolve** each unique link. A wikilink resolves if and only if **one** of the following is true:
   - A file exists at `wiki/entities/<text>.md`, `wiki/concepts/<text>.md`, or `wiki/synthesis/<text>.md` whose H1 matches `<text>` byte-for-byte.
   - Some page in `wiki/` has an `aliases:` frontmatter entry that matches `<text>` byte-for-byte.
   - The link uses explicit slug syntax (`[[slug|Display]]`) and `wiki/sources/slug.md` exists.

3. **Classify** each unresolved link:
   - **Must-create** — the link refers to an entity or concept the source clearly emphasizes (named in a section heading, discussed at length, or central to the source's claims). Create a stub page in the appropriate subdirectory with frontmatter, an H1 that matches the wikilink byte-for-byte, and a one-line description noting it was created as a stub during this ingest.
   - **Stub mention** — the link refers to something mentioned only in passing. Surface the list to the user and ask whether to create a stub page or strip the link from the source page.
   - **Typo or drift** — the link is a mistyped or recapitalized version of an existing page (e.g., `[[Deep dive]]` when the page is `Deep Dive`). Rewrite the link to match the existing page byte-for-byte.

4. **Re-run** the validation after fixes. Do not report success in step 8 until every wikilink resolves or has been explicitly approved as a stub. A clean ingest leaves zero broken links.

## Conventions

- Source summary pages are **factual only**. Save interpretation and synthesis for concept and synthesis pages.
- A single source typically touches **10-15 wiki pages**. This is normal and expected.
- When new information contradicts existing wiki content, **update the wiki page and note the contradiction** with both sources cited.
- **Prefer updating existing pages** over creating new ones. Only create a new page when the topic is distinct enough to warrant its own page.
- Use `[[wikilinks]]` for all internal references. Never use raw file paths.

## Wikilink Rules

The canonical rules for filenames, character normalization, and wikilink resolution live in your **agent config file** at the vault root (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, or `.cursor/rules/second-brain.mdc` depending on the agent). Read the **Page Naming** and **Character Normalization** sections there first — that file is the single source of truth for this vault.

The two rules that account for nearly all broken links you'll encounter:

1. **Exact-match resolution.** Every `[[wikilink]]` must match either a filename (entity/concept/synthesis pages) or an `aliases:` value (source pages) **byte-for-byte** — including capitalization, straight-vs-curly quotes, em-dashes, and ampersands. Step 9 validates this before declaring done.
2. **Source page link style.** Source-page filenames are kebab-case, but you never link by slug. Always link via the H1 (which the alias mirrors): `[[How to Build Agentic Pipelines (It's Simpler Than You Think)]]`, not `[[ben-fellows-how-to-build-pipelines]]`. The pipe form `[[slug|Display Text]]` is allowed when you need a different display text but otherwise unnecessary.

When you add a `## Sources` section to an entity or concept page, list each source by its H1 title as the wikilink — same rule as above.

## What's Next

After ingesting sources, the user can:
- **Ask questions** with `/second-brain-query` to explore what was ingested
- **Ingest more sources** — clip another article and run `/second-brain-ingest` again
- **Health-check** with `/second-brain-lint` after every 10 ingests to catch gaps
