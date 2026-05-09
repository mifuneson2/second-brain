# Wiki Schema

Canonical rules for LLM-maintained knowledge base wikis. This is the single source of truth — agent config templates pull from this document.

## Architecture

Three directories, three roles:

- **Clippings/** — immutable source documents. The LLM reads from here but NEVER modifies these files. Named to match the Obsidian Web Clipper default folder.
- **wiki/** — the LLM's workspace. Create, update, and maintain all files here.
- **output/** — reports, query results, and generated artifacts go here.

Wiki subdirectories:
- `wiki/sources/` — one summary page per ingested source
- `wiki/entities/` — pages for people, organizations, products, tools
- `wiki/concepts/` — pages for ideas, frameworks, theories, patterns
- `wiki/synthesis/` — comparisons, analyses, cross-cutting themes

Two special files:
- `wiki/index.md` — master catalog of every wiki page, organized by category. Update on every ingest.
- `wiki/log.md` — append-only chronological record. Never edit existing entries.

## Page Format

Every wiki page MUST include YAML frontmatter:

    ---
    tags: [tag1, tag2]
    sources: [source-filename-1.md, source-filename-2.md]
    created: YYYY-MM-DD
    updated: YYYY-MM-DD
    ---

Use `[[wikilink]]` syntax for all internal links. When you mention a concept, entity, or source that has its own page, link it.

## Operations

### Ingest (processing a new source)

When the user adds a file to Clippings/ and asks you to process it:

1. Read the source completely
2. Discuss key takeaways with the user
3. Create a source summary page in `wiki/sources/` with: title, source metadata, key claims, and a structured summary
4. Identify all entities and concepts mentioned. For each:
   - If a wiki page exists: update it with new information from this source, noting the source
   - If no wiki page exists: create one in the appropriate subdirectory
5. Add `[[wikilinks]]` between all related pages
6. Update `wiki/index.md` with any new pages
7. Append to `wiki/log.md`: `## [YYYY-MM-DD] ingest | Source Title`

A single source may touch 10-15 wiki pages. That is normal.

### Query (answering questions)

When the user asks a question:

1. Read `wiki/index.md` to find relevant pages
2. Read the relevant wiki pages
3. Synthesize an answer with `[[wikilink]]` citations to wiki pages
4. If the answer produces a valuable artifact (comparison, analysis, new connection), offer to save it as a new page in `wiki/synthesis/`
5. If you save a new page, update the index and log

### Lint (health check)

When the user asks you to lint or health-check the wiki:

1. Scan for contradictions between pages
2. Find stale claims that newer sources have superseded
3. Identify orphan pages (no inbound links)
4. Find important concepts mentioned but lacking their own page
5. Check for missing cross-references
6. Suggest data gaps that could be filled with a web search
7. Report findings and offer to fix issues
8. Log the lint pass: `## [YYYY-MM-DD] lint | Summary of findings`

## Index Format

Each entry in `wiki/index.md` is one line:

    - [[Page Name]] — one-line summary

Organized under category headers: Sources, Entities, Concepts, Synthesis.

## Log Format

Each entry in `wiki/log.md`:

    ## [YYYY-MM-DD] operation | Title
    Brief description of what was done.

## Page Naming

Two different conventions apply, depending on page type. Each convention is chosen so that `[[wikilinks]]` resolve **without an alias lookup whenever possible** — and where an alias is required (source pages), the rules below force exact-match aliases so resolution can't drift.

### Source pages (`wiki/sources/`)

Imported from external clippings (Obsidian Web Clipper, manual paste, etc.). The clipping filename is unfriendly (long, contains spaces, smart quotes, em-dashes), so source pages use a **slugified kebab-case filename + an `aliases:` field that matches the H1 byte-for-byte**.

- Filename: `wiki/sources/article-title-here.md`
- H1: `# Exact Article Title Here` (matches the original article title verbatim — preserves capitalization, em-dashes, ampersands)
- `aliases:` frontmatter: `["Exact Article Title Here"]` — must match the H1 byte-for-byte (see Character Normalization below)
- Wikilinks point to the alias: `[[Exact Article Title Here]]` — never `[[article-title-here]]`

The alias is what makes the wikilink resolve. Without it Obsidian creates an empty duplicate page at the vault root.

### Entity, concept, and synthesis pages

Authored by the LLM, not imported. The filename is the H1 — no slugification, no alias needed, because the wikilink resolves directly to the filename.

- `wiki/entities/Entity Name.md` → `# Entity Name` → `[[Entity Name]]`
- `wiki/concepts/Concept Name.md` → `# Concept Name` → `[[Concept Name]]`
- `wiki/synthesis/Comparison Topic.md` → `# Comparison Topic` → `[[Comparison Topic]]`

These filenames use Title Case with spaces. They follow the Character Normalization rules below — same character set as titles and aliases.

### Slugification rule for source-page filenames

Apply in order: lowercase → replace whitespace runs with single `-` → replace `&` with `and` → replace em-dash `—` and en-dash `–` with single `-` → drop characters not in `[a-z0-9-]` → collapse repeated `-` → trim leading/trailing `-` → truncate to 80 chars at a word boundary.

Example: `Dario Amodei — "We Are Near The End Of The Exponential"` → `dario-amodei-we-are-near-the-end-of-the-exponential`

## Character Normalization

These rules are the single source of truth. Every skill that reads, writes, or links to wiki pages MUST follow them. They apply to H1 titles, `aliases:` values, and the text inside `[[wikilinks]]`. (Filenames are governed by the slugification rule above for source pages, and by these same rules for entity/concept/synthesis pages.)

### Quotes

- **Allowed:** straight ASCII apostrophe `'` (U+0027) and straight ASCII double quote `"` (U+0022).
- **Never:** curly apostrophe `'` (U+2019), curly opening quote `"` (U+201C), curly closing quote `"` (U+201D), curly opening apostrophe `'` (U+2018).
- When you read source content that contains curly quotes, **substitute** to straight before writing the title or alias. Do this consistently in titles, aliases, and wikilinks — a wikilink with a curly quote will not resolve to an alias with a straight one, and Obsidian will spawn an empty duplicate page.

### Dashes

- **Em-dash `—` (U+2014):** preserved verbatim in titles, aliases, and wikilinks. Do not substitute hyphens — the source clipping uses em-dashes deliberately, and substituting breaks the alias match.
- **En-dash `–` (U+2013):** treated like em-dash — preserved verbatim.
- In source-page **filenames** (slugified), all dashes collapse to a single hyphen (see slugification rule).

### Ampersand

- **Titles, aliases, wikilinks:** kept verbatim as `&`.
- **Source-page filenames:** replaced with `and` during slugification.

### Forward slash, pipe, brackets, hash

- `/`, `|`, `[`, `]`, `#` are removed from titles entirely. Obsidian parses them specially: `|` is the wikilink alias separator, `[]` is wikilink syntax, `#` is heading-link syntax, `/` is folder-path. If a source's H1 contains them, drop them from the title and document the original in the body.

### Colon

- `:` is allowed in titles, aliases, and wikilinks.
- Removed from source-page **filenames** (filesystem-hostile on Windows).

### Capitalization (most common cause of broken links)

- The `aliases:` field must match the H1 **byte-for-byte** — every character, including case, punctuation, and whitespace.
- Every `[[wikilink]]` must match the alias (or, for entity/concept/synthesis pages, the filename) **byte-for-byte**.
- Never paraphrase, never recapitalize, never re-spell. If you mention `[[Dylan Patel — Deep Dive on TPU vs. Nvidia]]` in one page, every other page must use that exact string. "Deep dive" with a lowercase `d` will not resolve.

### Last-resort filename mangling (third tier)

Reserved for the rare case where a title contains characters that survive normalization but still can't appear in a filename on common filesystems. Apply this rule only after the above passes have been applied:

1. Strip these from the filename: `< > : " / \ | ? *` and ASCII control bytes 0x00–0x1F.
2. If the filename exceeds 200 bytes after stripping, truncate at the last word boundary that fits.
3. The H1 and `aliases:` keep the original characters — only the filename loses them.

This guarantees the alias remains the canonical link target while the filename stays portable.

## Image Handling

Web-clipped articles often include images. Handle them as follows:

1. **Download images locally.** In Obsidian Settings → Files and links, set "Attachment folder path" to `Clippings/assets/`. Then use "Download attachments for current file" (bind it to a hotkey like Ctrl+Shift+D) after clipping an article.
2. **Reference images from wiki pages** using standard markdown: `![description](../Clippings/assets/image-name.png)`. Keep the image in `Clippings/assets/` — never copy images into `wiki/`.
3. **During ingestion**, note any images in the source. If an image contains important information (diagrams, charts, data), describe its contents in the wiki page so the knowledge is captured in text form.

## Lint Frequency

Run a lint pass (`/second-brain-lint`) on this schedule:
- **After every 10 ingests** — catches cross-reference gaps while they're fresh
- **Monthly at minimum** — catches stale claims and orphan pages that accumulate over time
- **Before any major query or synthesis** — ensures the wiki is healthy before you rely on it for analysis

## Tools

You have access to these CLI tools — use them when appropriate:

- **summarize** — summarize links, files, and media. Run `summarize --help` for usage.
- **qmd** — local search engine for markdown files. Run `qmd --help` for usage. Use when the wiki grows beyond what index.md can efficiently navigate.
- **agent-browser** — browser automation for web research. Use when web_search or web_fetch fail.

## Rules

1. Never modify files in `Clippings/`. They are immutable source material.
2. Always update `wiki/index.md` when you create or delete a page.
3. Always append to `wiki/log.md` when you perform an operation.
4. Use `[[wikilinks]]` for all internal references. Never use raw file paths in page content.
5. Every wiki page must have YAML frontmatter with tags, sources, created, and updated fields.
6. When new information contradicts existing wiki content, update the wiki page and note the contradiction with both sources cited.
7. Keep source summary pages factual. Save interpretation and synthesis for concept and synthesis pages.
8. When asked a question, search the wiki first. Only go to raw sources if the wiki doesn't have the answer.
9. Prefer updating existing pages over creating new ones. Only create a new page when the topic is distinct enough to warrant it.
10. Keep `wiki/index.md` concise — one line per page, under 120 characters per entry.
