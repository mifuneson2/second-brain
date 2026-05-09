---
name: second-brain-rollback
description: >
  Read the latest auto-ingest review file (or one specified by date),
  execute rollbacks for every ticked decision, and append every ticked
  policy suggestion to wiki/auto-ingest-policies.md. Use after reviewing
  the morning's /second-brain-ingest-auto results to undo bad calls and
  teach the skill for next time. Args: optional date (YYYY-MM-DD) of
  the review to process; defaults to most recent review file in output/.
allowed-tools: Bash Read Write Edit Glob Grep
---

# Second Brain — Rollback

Companion skill to `/second-brain-ingest-auto`. Reads a daily auto-ingest review file (`output/auto-ingest-review-{date}.md`), parses Obsidian-style markdown checkboxes (`- [ ]` vs `- [x]`), and executes:

1. **Rollbacks** for every ticked decision — undoing the auto-skill's choice using the `rollback_steps` recipe stored in `output/decisions-{date}.json`.
2. **Policy promotions** for every ticked policy suggestion — appending the rule to `wiki/auto-ingest-policies.md` so future auto-ingest runs honor it.

## When to use

Run after eyeballing the daily review file and ticking what you want changed:
- Tick a decision's "Roll back this decision" checkbox if the auto-skill made the wrong call.
- Tick a policy-suggestion checkbox if the rule should apply going forward.

Then invoke this skill. By default it reads the most recent review file in `output/` (the one not yet moved to `output/processed/`). Pass a date to target a specific one: `/second-brain-rollback 2026-05-09`.

## Workflow

### 1. Locate the review file

- If the user provided a date arg, target `output/auto-ingest-review-{date}.md`.
- Otherwise, find the most recent `output/auto-ingest-review-*.md` (by filename date) that has not been moved to `output/processed/`.
- If no review file is found, tell the user and stop.

### 2. Parse ticked checkboxes

Read the review file. Identify two kinds of ticked lines:

**Rollback ticks** — lines starting with `- [x]` that appear under a `### Decision dN` heading. Extract:
- The decision ID (e.g., `d1`, `d2`) from the most recent heading above the ticked line.
- A flag indicating this is a rollback request.

```bash
# Example pattern: walk the file once, track the current ### Decision dN heading,
# and when a `- [x]` line appears, attribute it to that decision ID.
```

**Policy ticks** — lines starting with `- [x]` that appear under the `## Suggested policy entries` heading. Extract:
- The full bullet line text (everything after `- [x] `) — that's the proposed rule.
- The "section" hint from the corresponding `policy_suggestions` entry in the JSON log (e.g., "Fuzzy-match overrides").

If a `- [x]` line is in both a `### Decision dN` and the `## Suggested policy entries` section, it's a parser bug; treat as the section it appears in based on document order.

### 3. Load the decision log

Read `output/decisions-{date}.json` matching the review file's date. For each ticked rollback ID, locate the corresponding `decisions[]` entry and pull its `rollback_steps` array.

If the JSON is missing or malformed, abort with a clear error — never half-apply rollbacks.

### 4. Execute rollback steps

For each ticked decision, execute its `rollback_steps` in order. Dispatch by `op`:

| `op` | Implementation |
|---|---|
| `rewrite_link` | In `file`, replace exactly one occurrence of `from` with `to`. Use a literal find-and-replace, not regex. If `from` doesn't appear, log a warning (link may have moved) and continue. |
| `create_stub` | Create the file at `path` with frontmatter (`tags: [stub]`, today's date for `created`/`updated`), an H1 set to `h1`, and a one-line body containing `context_line` (or "Auto-recreated during rollback on YYYY-MM-DD" if missing). Add an index entry to `wiki/index.md` under the appropriate category. |
| `delete_file` | Delete `path`. If the file doesn't exist, log a warning and continue. |
| `remove_index_entry` | In `wiki/index.md`, find the first line whose prefix matches `match` and delete it. |
| `restore_link_at_offset` | In `file`, insert `text` at the byte `offset` recorded at strip time. If the file's content has drifted enough that the offset doesn't make sense (the surrounding bytes don't match what was captured at log time), fall back to appending the link to the end of the relevant section and log a warning. |
| `update_h1_and_alias` | Prompt the user (since this is the only op that needs new input from a human): ask for the corrected H1, set the file's H1 and `aliases:` to that value. |

After all `rollback_steps` for a single decision succeed, mark the decision in the JSON as `"rolled_back": true` (mutate the in-memory copy; you'll write the updated JSON back when archiving the review).

### 5. Append ticked policies

For each ticked policy suggestion:

1. Look up the corresponding `policy_suggestions[]` entry in the JSON to get the `section` ("Fuzzy-match overrides", "Stub-creation rules", "Title derivation", or "Other").
2. Open `wiki/auto-ingest-policies.md`. If it doesn't exist, create it from the template in the auto skill's SKILL.md.
3. Append the rule as a bullet under the matching section. If the section header doesn't exist, append a new section.

### 6. Re-validate wikilinks

After rollbacks have changed wiki contents, run the same wikilink-validation logic as `/second-brain-ingest` Step 9 across the touched files. Any new broken links surface in the rollback summary so the user knows to clean up. The rollback skill itself does NOT auto-create stubs to fix newly-broken links — that's the point of the user reverting.

### 7. Archive the review file

Move `output/auto-ingest-review-{date}.md` and `output/decisions-{date}.json` to `output/processed/` so the next `/second-brain-rollback` invocation doesn't double-apply the same ticks. Use `mkdir -p output/processed/` first if needed.

### 8. Log

Append to `wiki/log.md`:

```markdown
## [YYYY-MM-DD] rollback | N decisions reverted, M policies added
Reverted decisions: d1 (fuzzy_link_rewrite), d3 (stub_create).
Policies added to auto-ingest-policies.md: "Never auto-merge CoreWave with CoreWeave".
Touched files: ...
Newly broken wikilinks (from rollback): ...
```

### 9. Report to user

Tell the user what was done:
- Decisions reverted (with brief description of each).
- Policies added (with the rule text).
- Any new broken wikilinks introduced by the rollback (will need manual fixup).
- Confirmation that the review file is archived to `output/processed/`.

## Idempotency

This skill is safe to re-run on the same date — but only if the review file is still in `output/`. Once archived to `output/processed/`, a re-run for that date will report "no review file found." If you need to apply additional ticks to an already-archived review, move the file back to `output/` first.

## Edge cases

- **JSON log missing**: error out. Tell the user the review file references decisions that have no rollback recipe; suggest manual fixup.
- **A `rollback_steps` op fails partway** (e.g., file already deleted, link not found): log the failure but continue with remaining steps — partial rollback is better than none.
- **A ticked decision has already been rolled back** (the JSON's `rolled_back: true` flag is set): skip it, note in the report. Should only happen if someone moved a processed review file back manually.
- **Policy file already contains a near-duplicate rule**: append anyway (deduping is the user's job; lint can flag dupes later). Don't try to be clever about merging.
- **Date arg points at a non-existent review**: list available review files (in `output/` and `output/processed/`) and stop.

## What's Next

After running this skill:
- The rolled-back decisions' artifacts are gone (or restored, in the strip case).
- Future `/second-brain-ingest-auto` runs will read the updated policy file and behave accordingly.
- Run `/second-brain-lint` if rollbacks introduced broken wikilinks you want to clean up systematically.

## Related Skills

- `/second-brain-ingest-auto` — produces the review files this skill consumes.
- `/second-brain-ingest` — interactive ingest; doesn't write rollback logs.
- `/second-brain-lint` — useful after rollbacks to catch newly-broken wikilinks.
