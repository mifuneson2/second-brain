---
name: second-brain-ingest-auto
description: >
  Non-interactive variant of /second-brain-ingest for scheduled or
  unattended runs. Processes all unprocessed files in Clippings/
  silently, applies aggressive defaults to ambiguous decisions, writes
  a JSON decision log and a markdown review file to output/, and
  appends a summary to wiki/log.md. Use when invoked by /schedule,
  /loop, cron, or any routine without a human in the loop. Use
  /second-brain-ingest instead when a human can review choices live.
allowed-tools: Bash Read Write Edit Glob Grep
---

# Second Brain — Ingest (Auto)

Non-interactive variant of `/second-brain-ingest` for scheduled / unattended runs. Same workflow, same wiki schema, same Character Normalization rules — but every interactive checkpoint is replaced by an aggressive default that gets logged for morning review.

## When to use this skill vs `/second-brain-ingest`

- **`/second-brain-ingest`** — human at the keyboard. Will pause for takeaways, fuzzy-match confirmation, stub-vs-strip decisions.
- **`/second-brain-ingest-auto`** (this skill) — scheduled cron/routine, `/loop`, or any context without a human. Decides everything itself, logs decisions, and produces a review file you read in the morning to accept/roll back the auto-decisions.

If you can't tell which mode you're in: assume interactive. The auto skill should only fire when the prompt explicitly says scheduled / unattended.

## Inputs

- The vault root (working directory).
- Optional: `wiki/auto-ingest-policies.md` — user-curated overrides. Read on every run; treated as authoritative when in conflict with the skill's defaults.

## Workflow

The auto skill follows the same nine-step ingest workflow as `/second-brain-ingest` (see that skill's SKILL.md and your agent config's Page Naming + Character Normalization sections — same canonical rules apply). The four interactive steps are replaced as follows:

| Step | Interactive (`/second-brain-ingest`) | Auto (this skill) |
|---|---|---|
| 0 (no H1 case) | Ask user to confirm derived title | Derive from filename, log as `title_derived` decision |
| 2 (key takeaways) | Discuss with user, await confirmation | **Skip entirely** |
| 4 (fuzzy match) | Ask user to merge into existing | Rewrite the wikilink in the source page to point at the existing entity. Do **not** create the new page. Do **not** mix new content into the existing page. Log as `fuzzy_link_rewrite`. |
| 9 (unresolved links) | Ask must-create vs strip | Auto-classify by heuristic. Log each as `stub_create` or `stub_strip`. |

Steps 1, 3, 5, 6, 7, 8 are unchanged from `/second-brain-ingest`.

### Step 0 (auto): Identify the canonical title

1. Open the source. Find the first H1.
2. Apply Character Normalization (see your agent config).
3. **If no H1 exists**: derive a title by reverse-slugifying the filename (`_` and `-` → spaces, drop `.md`, apply title case). Record this as a `title_derived` decision. Use the derived title as the source page's H1 and `aliases:` value. Continue.
4. Hold the normalized title in working memory for use in steps 3, 5, and 9.

### Step 4 (auto): Update entity and concept pages

For each entity or concept mentioned in the source:

1. **Search for fuzzy match** across existing entity/concept/synthesis pages (case-insensitive substring; common variants like Marc/Mark, OpenAI/Open AI, Google DeepMind/Google+DeepMind; Levenshtein ≤ 2 on full names).
2. **Apply the policy file first** (`wiki/auto-ingest-policies.md`). If a policy says "Never auto-merge X with Y", honor it — create a new page or stub instead.
3. **If a fuzzy match remains and no policy blocks it**: rewrite the wikilink in the source page from `[[NewMention]]` to `[[ExistingEntity]]`. Do NOT modify the existing entity page. Log as `fuzzy_link_rewrite` decision. The source page itself carries the new context (in its summary, key claims, and entity-mention list).
4. **If the existing wiki page is an exact match**: update it normally — append source to `sources:` frontmatter, add new information, update `updated:` date.
5. **If no match exists**: create a new page in the appropriate subdirectory with proper frontmatter. (This is "content added", not a logged decision — there's no ambiguity.)

### Step 9 (auto): Validate wikilinks before declaring done

Same collection step as `/second-brain-ingest`. For each unresolved wikilink, classify automatically:

**Heuristic for `stub_create` vs `stub_strip`**, in priority order:

1. If the policy file has a rule, follow it.
2. **`stub_create`** if any of:
   - The link appears in a section heading (`## See Also`, `## Sources`, etc.) of the source page.
   - The link appears 3+ times across all pages touched in this run.
   - The link target is a proper noun ending in Inc/Corp/LLC/Co (likely an entity).
3. Otherwise → **`stub_strip`** — remove the wikilink from the source page (keep the surrounding text), log as `stub_strip`.

For each `stub_create`: create a minimal page in the appropriate subdirectory with frontmatter, an H1 matching the wikilink byte-for-byte, and a one-line description noting it was auto-created on the run date. Add an entry to `wiki/index.md`.

For each `stub_strip`: edit the source page to remove the `[[wikilink]]` markup, capturing the byte offset for rollback. The surrounding prose stays intact.

After fixes, re-validate. Do not finish the file's processing until every wikilink resolves.

## Policy file

Read `wiki/auto-ingest-policies.md` at the start of every run. If it doesn't exist, create it with the template at the end of this skill.

The policy file is plain-language rules the user has accumulated over time. Examples:

- "Never auto-merge `Marc` and `Mark` — they are different people."
- "Always create stubs for entities named in `## See Also` headings."
- "When deriving titles from filenames starting with a date prefix like `04-07_`, drop the prefix."

Treat each rule as authoritative for the kind of decision it describes. When a rule conflicts with the default heuristic, the rule wins. Log in the run's JSON which policies were consulted and which fired.

## Outputs

After processing all unprocessed files, write three artifacts:

### 1. JSON decision log

Path: `output/decisions-{YYYY-MM-DD}.json` (one per calendar date; if a file already exists for today, merge into it under a new `run_id`).

```json
{
  "run_id": "2026-05-09T03:00:00Z",
  "files_processed": [
    {
      "clipping": "Clippings/Jensen Huang — TPU Competition...md",
      "source_page": "wiki/sources/jensen-huang-tpu-competition.md"
    }
  ],
  "decisions": [
    {
      "id": "d1",
      "type": "fuzzy_link_rewrite",
      "context_source_page": "wiki/sources/jensen-huang-tpu-competition.md",
      "trigger": "Source mentioned 'CoreWave'; existing entity 'CoreWeave' (Levenshtein 1)",
      "action": {
        "rewrote_link_in": "wiki/sources/jensen-huang-tpu-competition.md",
        "from": "[[CoreWave]]",
        "to": "[[CoreWeave]]"
      },
      "rollback_steps": [
        { "op": "rewrite_link", "file": "wiki/sources/jensen-huang-tpu-competition.md", "from": "[[CoreWeave]]", "to": "[[CoreWave]]" },
        { "op": "create_stub", "path": "wiki/entities/CoreWave.md", "h1": "CoreWave", "context_line": "..." }
      ]
    },
    {
      "id": "d2",
      "type": "stub_create",
      "context_source_page": "wiki/sources/dario-amodei-end-of-exponential.md",
      "trigger": "Wikilink [[OpenAI]] unresolved; classified must-create (named in heading)",
      "action": {
        "created_path": "wiki/entities/OpenAI.md",
        "h1": "OpenAI",
        "added_index_entry": "- [[OpenAI]] — Frontier AI lab (stub)"
      },
      "rollback_steps": [
        { "op": "delete_file", "path": "wiki/entities/OpenAI.md" },
        { "op": "remove_index_entry", "file": "wiki/index.md", "match": "- [[OpenAI]] —" }
      ]
    },
    {
      "id": "d3",
      "type": "stub_strip",
      "context_source_page": "wiki/sources/some-page.md",
      "trigger": "Wikilink [[Random Mention]] unresolved; classified passing-mention",
      "action": {
        "stripped_link": "[[Random Mention]]",
        "from_file": "wiki/sources/some-page.md",
        "byte_offset": 4823
      },
      "rollback_steps": [
        { "op": "restore_link_at_offset", "file": "wiki/sources/some-page.md", "offset": 4823, "text": "[[Random Mention]]" }
      ]
    },
    {
      "id": "d4",
      "type": "title_derived",
      "context_source_page": "wiki/sources/some-untitled-clipping.md",
      "trigger": "Source had no H1; derived from filename 'some_untitled_clipping.md'",
      "action": {
        "derived_h1": "Some Untitled Clipping",
        "set_aliases": ["Some Untitled Clipping"]
      },
      "rollback_steps": [
        { "op": "update_h1_and_alias", "file": "wiki/sources/some-untitled-clipping.md", "new_h1": "<USER_PROVIDED>", "new_aliases": ["<USER_PROVIDED>"] }
      ]
    }
  ],
  "stats": {
    "files_processed": 1,
    "pages_created": 4,
    "pages_updated": 6,
    "fuzzy_link_rewrites": 1,
    "stubs_created": 1,
    "stubs_stripped": 1,
    "titles_derived": 1
  },
  "policy_suggestions": [
    {
      "id": "p1",
      "rule": "Never auto-merge \"CoreWave\" with \"CoreWeave\" — comes up again, ask first",
      "section": "Fuzzy-match overrides",
      "based_on": ["d1"]
    }
  ]
}
```

Each `rollback_steps` entry uses one `op` from the dispatch table consumed by `/second-brain-rollback`:

| `op` | Required fields | What it does |
|---|---|---|
| `rewrite_link` | `file`, `from`, `to` | Replace `from` with `to` in the file. |
| `create_stub` | `path`, `h1`, `context_line` | Create a minimal entity/concept page. |
| `delete_file` | `path` | Remove the file. |
| `remove_index_entry` | `file`, `match` | Delete a line in `wiki/index.md` matching the prefix. |
| `restore_link_at_offset` | `file`, `offset`, `text` | Re-insert the wikilink at the given byte offset. |
| `update_h1_and_alias` | `file`, `new_h1`, `new_aliases` | Set H1 and `aliases:` (called manually if user wants to override the derived title). |

### 2. Markdown review file

Path: `output/auto-ingest-review-{YYYY-MM-DD}.md`. Format:

```markdown
# Auto-Ingest Review — 2026-05-09

Processed 1 file, made 4 decisions.

To roll back any decision or accept any policy suggestion, tick its
checkbox (Obsidian: click the box) and then run `/second-brain-rollback`.

## Files processed

- ✓ Clippings/Jensen Huang — TPU Competition...md → [[Jensen Huang — TPU Competition, ...]] (4 decisions)

## Decisions

### Decision d1 — fuzzy-link-rewrite: `[[CoreWave]]` → `[[CoreWeave]]`

**Source:** [[Jensen Huang — TPU Competition, ...]]
**Why:** Existing entity `CoreWeave` matched `CoreWave` (Levenshtein 1).
**What I did:** Rewrote the wikilink in the source page from `[[CoreWave]]` to `[[CoreWeave]]`. Did not modify the existing `wiki/entities/CoreWeave.md`.

- [ ] Roll back this decision (rewrites link back to `[[CoreWave]]`, creates a new stub `wiki/entities/CoreWave.md`)

### Decision d2 — stub-create: [[OpenAI]]

**Source:** [[Dario Amodei — End of Exponential]]
**Why:** Wikilink `[[OpenAI]]` had no resolution; appeared in a `## See Also` section so classified must-create.
**What I did:** Created `wiki/entities/OpenAI.md` with frontmatter + one-line description; added an index entry.

- [ ] Roll back this decision (deletes the stub file, removes the index entry; leaves `[[OpenAI]]` link as-is in the source — it will show as broken)

### Decision d3 — stub-strip: [[Random Mention]]

**Source:** [[Some Page]]
**Why:** Wikilink `[[Random Mention]]` unresolved, appeared once in body prose only — classified passing mention.
**What I did:** Removed the `[[Random Mention]]` markup from the source page; surrounding text intact.

- [ ] Roll back this decision (re-inserts `[[Random Mention]]` at byte offset 4823)

### Decision d4 — title-derived: "Some Untitled Clipping"

**Source:** wiki/sources/some-untitled-clipping.md
**Why:** Source file had no H1. Derived from filename `some_untitled_clipping.md`.
**What I did:** Set H1 and `aliases:` to "Some Untitled Clipping".

- [ ] Roll back this decision (you'll be prompted for the correct title)

## Suggested policy entries

Tick any that match how you'd want the skill to behave going forward. Ticked ones get appended to `wiki/auto-ingest-policies.md` when you run `/second-brain-rollback`.

- [ ] **Never auto-merge `CoreWave` with `CoreWeave`** — comes up again, ask first

## How to apply

After ticking the boxes above, run `/second-brain-rollback 2026-05-09`
(or just `/second-brain-rollback` for the most recent review).
```

### 3. Log entry

Append to `wiki/log.md`:

```markdown
## [YYYY-MM-DD] auto-ingest | N files, M decisions
Processed N files. Decisions: K fuzzy-link rewrites, L stubs created,
M stubs stripped, P titles derived. Review at output/auto-ingest-review-{date}.md.
```

## Edge cases

- **Zero unprocessed files**: write a minimal review file noting "no work" and skip the JSON. Don't pollute `output/` with empty logs. Skip the wiki/log.md append.
- **Source file unreadable / corrupt**: skip it, log a `failed_files` entry in the JSON, surface in review under a "Failures" section, continue with remaining files.
- **A wikilink would resolve only if you applied a policy rule that doesn't exist yet**: still apply the safest default (e.g., create stub), and emit a policy suggestion for it.
- **Multiple runs in a single day**: append a new `run_id` block to the existing `decisions-{date}.json`. Append a new section to the review file. Don't clobber.
- **A run is interrupted mid-file**: leave the partial state in place but don't claim the file was processed in `wiki/log.md`. Next run will retry the file.

## `wiki/auto-ingest-policies.md` template

Created on first run if missing:

```markdown
# Auto-Ingest Policies

Rules that override `/second-brain-ingest-auto`'s aggressive defaults.
Read at the start of every auto-ingest run. Hand-curated by you;
appended to by `/second-brain-rollback` when you tick policy
suggestions in daily review files.

## Fuzzy-match overrides

(Rules that block or force specific name-variant merges. Example:
"Never auto-merge `Marc` and `Mark` — they are different people.")

## Stub-creation rules

(Rules tightening or loosening the stub_create vs stub_strip heuristic.
Example: "Always create a stub for entities named in `## See Also`.")

## Title derivation

(Rules for deriving titles from filenames. Example: "When the filename
starts with `MM-DD_`, drop the date prefix before slugifying.")

## Other

(Anything else you want the skill to know about your wiki conventions.)
```

## What's Next

After the auto skill finishes a run, the user reviews `output/auto-ingest-review-{date}.md` in Obsidian, ticks rollback boxes for any wrong calls, ticks policy boxes for rules they want enforced going forward, and runs `/second-brain-rollback` to apply both.

Schedule the auto skill via `/schedule` (Claude Code remote routines) or `/loop` for a recurring local cadence.
