# Second Brain

An LLM-maintained personal knowledge base built on the [LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). Drop raw sources into a folder, let the LLM compile them into a structured wiki, and browse it all in Obsidian.

![Second Brain Overview](docs/assets/second-brain-overview.png)

## How It Works

You feed raw material (articles, papers, notes, transcripts) into a `Clippings/` folder (named to match the Obsidian Web Clipper default). The LLM reads everything, writes structured wiki pages, creates cross-references, and maintains an index. You browse the results in Obsidian — following links, exploring the graph view, and asking questions.

The LLM is the librarian. You're the curator.

## Prerequisites

- **[Obsidian](https://obsidian.md)** — the markdown editor you'll browse your wiki in
- **An AI coding agent** — [Claude Code](https://claude.ai/code), [Codex](https://openai.com/codex), [Cursor](https://cursor.com), [Gemini CLI](https://github.com/google-gemini/gemini-cli), or any agent that supports [Agent Skills](https://agentskills.io)
- **[Node.js](https://nodejs.org)** — required for installing the skills via npm

## Install

```bash
npx skills add NicholasSpisak/second-brain
```

This installs four skills into your AI agent (Claude Code, Codex, Cursor, Gemini CLI, and 40+ others):

| Skill | What it does |
|---|---|
| `/second-brain` | Set up a new vault (guided wizard) |
| `/second-brain-ingest` | Process raw sources into wiki pages |
| `/second-brain-query` | Ask questions against your wiki |
| `/second-brain-lint` | Health-check the wiki |

## Quick Start

1. **Install the skills** (see above)
2. **Run the wizard:** type `/second-brain` in your AI agent — it walks you through naming, location, domain, and tooling
3. **Install Web Clipper:** [Obsidian Web Clipper](https://chromewebstore.google.com/detail/obsidian-web-clipper/cnjifjpddelmedmihgijeibhnjfabmlf) — configure it to save to your vault's `Clippings/` folder
4. **Open in Obsidian** — launch Obsidian, choose "Open folder as vault", select your vault folder
5. **Clip your first article** to `Clippings/`, then run `/second-brain-ingest` — the LLM will discuss key takeaways and build wiki pages
6. **Browse your wiki** in Obsidian — follow `[[wikilinks]]`, explore the graph view, check `wiki/index.md`
7. **Keep going** — `/second-brain-query` to ask questions, `/second-brain-lint` to health-check

## What You Get

```
your-vault/
├── Clippings/              # Your inbox — drop sources here (Obsidian Web Clipper default)
│   └── assets/             # Images and attachments
├── wiki/                   # LLM-maintained wiki
│   ├── sources/            # One summary per ingested source
│   ├── entities/           # People, orgs, products, tools
│   ├── concepts/           # Ideas, frameworks, theories
│   ├── synthesis/          # Comparisons, analyses, themes
│   ├── index.md            # Master catalog of all pages
│   └── log.md              # Chronological operation record
├── output/                 # Reports and generated artifacts
└── CLAUDE.md               # Agent config (varies by agent)
```

## Optional Tools

The wizard offers to install these. All optional but recommended:

- **[summarize](https://github.com/steipete/summarize)** — summarize links, files, and media from the CLI
- **[qmd](https://github.com/tobi/qmd)** — local search engine for markdown files (useful as wiki grows)
- **[agent-browser](https://github.com/vercel-labs/agent-browser)** — browser automation for web research

## FAQ

**The wizard failed or I need to re-run setup.**
Run `/second-brain` again — the onboarding script is idempotent. It won't overwrite existing files, so your data is safe. If you need a fresh start, delete the vault folder and re-run.

**I accidentally modified a file in `Clippings/`.**
That's OK. The wiki was built from the original content. If you need the original back, check your git history (if the vault is a git repo) or re-clip the source. The wiki pages are unaffected.

**`wiki/index.md` is out of sync with actual pages.**
Run `/second-brain-lint` — it checks index consistency and offers to fix mismatches.

**Wikilinks are broken after renaming a page.**
Run `/second-brain-lint` — it scans for broken `[[wikilinks]]` and reports which files need updating.

**The wiki is getting large and queries are slow.**
Install `qmd` (`npm i -g @tobilu/qmd`). The query skill uses it automatically when available. It provides fast hybrid search across your wiki files.

**Can I use this with multiple AI agents?**
Yes. The wizard generates config files for each agent you select. They all follow the same wiki schema, so multiple agents can work on the same vault.

**How do I handle images in clipped articles?**
In Obsidian, set Settings → Files and links → Attachment folder path to `Clippings/assets/`. After clipping an article, use "Download attachments for current file" to save images locally.

**How often should I lint?**
After every 10 ingests or monthly — whichever comes first. Also run it before any major query or synthesis work.

## Based On

- [Andrej Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [Agent Skills open standard](https://agentskills.io)
- [Blueprint & origin story](docs/REQUIREMENTS.md) — the founding document for this project

---

<p align="center">Want to learn how to build projects like this with AI? Join the <a href="https://www.skool.com/buildwithai/about">Build With AI</a> community.</p>
