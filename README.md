# LLM Wiki

A pattern for building personal knowledge bases with LLMs. The LLM incrementally builds and maintains a structured, interlinked wiki from your raw sources — instead of re-deriving answers from scratch on every query.

Designed for [pi](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent) + [Obsidian](https://obsidian.md), but the idea is agent-agnostic.

## How it works

You curate raw sources (articles, papers, notes). The LLM reads them, extracts key information, and integrates it into a persistent wiki — creating entity pages, concept summaries, cross-references, and flagging contradictions. Knowledge compounds with every source you add.

Three layers:

- **`raw/`** — your source documents (immutable, LLM never modifies)
- **`wiki/`** — LLM-generated markdown pages (index, entities, concepts, synthesis, etc.)
- **`AGENTS.md`** — schema that tells the LLM how to structure and maintain the wiki

## Setup

```bash
git clone <this-repo> ~/wiki
cd ~/wiki
pi
```

Open the same directory as an Obsidian vault to browse wiki pages, follow wikilinks, and view the knowledge graph in real time.

## Skills

Three pi skills are included in `.pi/skills/`:

| Skill | Usage | Description |
|-------|-------|-------------|
| ingest | `/skill:ingest raw/articles/xxx.md` | Process a source and integrate into the wiki |
| query | `/skill:query your question here` | Answer questions using the wiki |
| lint | `/skill:lint` | Health-check: find orphans, broken links, contradictions |

## Directory structure

```
├── AGENTS.md              # Wiki schema and conventions
├── .pi/skills/            # Pi skills (ingest, query, lint)
├── raw/                   # Raw sources (articles, papers, notes, assets)
└── wiki/                  # LLM-maintained knowledge base
    ├── index.md           # Content index
    ├── log.md             # Operation log
    ├── entities/          # People, orgs, products
    ├── concepts/          # Ideas, theories, topics
    ├── sources/           # Per-source summaries
    ├── comparisons/       # Side-by-side analyses
    └── synthesis/         # Cross-source insights
```

## Adapting to your domain

Edit `AGENTS.md` to adjust categories, page format, and conventions for your use case. The skills in `.pi/skills/` can also be modified to change the ingest/query/lint workflows.

## Credits

Based on the [LLM Wiki](https://github.com/tobi/llm-wiki) pattern by Tobi Lutke.

## License

MIT
