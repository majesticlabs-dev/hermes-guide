# GBrain Memory Plugin

GBrain is a searchable Hermes pull-memory provider. The active deployment may use a registered canonical source and different storage engines. The SQLite/FTS5 details in this page describe the legacy standalone plugin architecture, not a universal storage contract.

It is inspired by [garrytan/gbrain](https://github.com/garrytan/gbrain), but the Hermes plugin is intentionally boring infrastructure: deterministic entity extraction and graph-style linking between related notes.

> **Current memory-routing rule:** Keep `MEMORY.md` and `USER.md` for every-turn steering. Put durable low-frequency profile context into a profile-specific GBrain `hermes-memory` page, such as `hermes/profile-memory-overflow/<profile>`. Do not create a profile-local overflow file. The active GBrain source is canonical and must be discovered with `gbrain sources list`.

## Where to host it

Use two homes:

1. **Plugin source code:** the Hermes Agent plugin tree or the standalone public plugin repo.
   - Standalone repo: [majesticlabs-dev/hermes-gbrain-plugin](https://github.com/majesticlabs-dev/hermes-gbrain-plugin)
   - Canonical bundled location: `plugins/memory/gbrain/`
   - Why: GBrain implements the Hermes `MemoryProvider` interface, uses Hermes plugin registration, adds the `gbrain_note` tool, and needs to be tested with Hermes' memory lifecycle.
   - Best long-term home: upstream Hermes Agent if the plugin is intended for all Hermes users.

2. **Operator guide / runbook:** this guide repo.
   - Canonical docs file: `gbrain-memory-plugin.md`
   - Why: the guide should explain when to use it, how to enable it, what it stores, and the operational tradeoffs.

Do **not** host the plugin code only in this guide repo. A docs repo is the wrong home for executable Hermes provider code. The current standalone public source repo is [majesticlabs-dev/hermes-gbrain-plugin](https://github.com/majesticlabs-dev/hermes-gbrain-plugin); this guide documents the install path and operating model.

## What it does

GBrain stores explicit notes, extracts entities from those notes, and links notes that share entities.

Extracted entity types:

- URLs: `https://example.com`
- file paths: `src/app.py`, `~/.config/hermes/config.yaml`
- handles: `@alice`
- tags: `#python`
- quoted phrases: `"launch checklist"`
- capitalized names: `Project Alpha`, `New York`
- aliases: `Robert aka Bob`, `X also known as Y`

Core capabilities:

- **Local SQLite store:** data lives under the active Hermes home/profile.
- **FTS5 search:** full-text search when SQLite FTS5 is available.
- **LIKE fallback:** search still works on SQLite builds without FTS5.
- **Entity graph:** notes with shared entities can be retrieved as linked context.
- **Prefetch recall:** relevant notes can be injected before a turn.
- **Memory mirroring:** explicit Hermes memory writes can be mirrored into GBrain through the memory provider hook.

## Storage model

For the active GBrain deployment, the registered source owns the canonical files and database. Discover the source instead of assuming a per-profile SQLite path:

```bash
gbrain sources list
```

Profile-specific compaction pages should be written with a stable slug such as `hermes/profile-memory-overflow/<profile>` and verified with `gbrain get`. The `~/.hermes/gbrain/gbrain.db` paths below apply only to the legacy standalone plugin architecture described by this page. They are not the default storage contract for the current GBrain CLI.

## Enable it

Interactive setup:

```bash
hermes memory setup
# choose: gbrain
```

Manual config:

```bash
hermes config set memory.provider gbrain
```

Then start a fresh Hermes session or restart the gateway so the provider is loaded.

## Tool: `gbrain_note`

The provider exposes one tool with four actions.

### Add a note

```python
gbrain_note(action="add", content="Project Alpha uses #python and ships from src/app.py")
```

Returns:

```json
{
  "note_id": 1,
  "entities": {
    "urls": [],
    "file_paths": ["src/app.py"],
    "handles": [],
    "tags": ["python"],
    "quoted": [],
    "capped": ["Project Alpha"],
    "aliases": []
  },
  "aliases": []
}
```

### Search notes

```python
gbrain_note(action="search", query="Project Alpha", limit=5)
```

Returns matching notes, ordered by FTS relevance when FTS5 is available.

### Find linked notes

By note ID:

```python
gbrain_note(action="links", note_id=1, limit=10)
```

By entity:

```python
gbrain_note(action="links", entity="python", limit=10)
```

This returns notes that share the same extracted entity.

### Stats

```python
gbrain_note(action="stats")
```

Returns store counts such as total notes, total entities, total aliases, and database path.

## Backfill existing Hermes memory

If a Hermes install already has `MEMORY.md` and `USER.md` entries, backfill them into GBrain with the migration script from the Hermes source tree.

Dry run first:

```bash
python scripts/gbrain_backfill.py --dry-run
```

Run the backfill:

```bash
python scripts/gbrain_backfill.py
```

Custom paths:

```bash
python scripts/gbrain_backfill.py \
  --hermes-home ~/.hermes \
  --db-path ~/.hermes/gbrain/gbrain.db
```

The backfill script should not modify the source markdown memory files. It reads entries and writes deduplicated notes into the GBrain database.

## When to use GBrain

Use GBrain when you want:

- local-only memory with no external vendor dependency
- searchable notes that preserve exact user-written content
- lightweight graph recall without embeddings
- entity-based linking across projects, people, URLs, paths, and tags
- deterministic behavior that is easy to inspect and test
- a personal AI partner that can recall broad context without putting every fact into the prompt

Do not use GBrain as a replacement for:

- long-form documents or a personal knowledge base
- scoped team/project wikis with explicit access boundaries
- vector semantic search over large corpora
- confidential secret storage
- arbitrary session transcript dumping

GBrain is best for durable notes, explicit memory facts, and recall across many small pieces of context. A wiki or knowledge base remains the better home for authored documents, project operating state, specs, decision logs, and material a team member should be able to read directly.

## GBrain vs scoped wikis

For agent-first teams, avoid one giant shared memory blob. Use scope boundaries:

- **Personal GBrain:** broad personal recall for the user's private partner/chief-of-staff agent. It may know books read, mentors, preferences, decisions, contacts, and cross-domain context.
- **Personal knowledge base / wiki:** authored durable knowledge the user wants to own across machines: ideas, learnings, strategy, project notes, templates, and decision history.
- **Scoped team or topic wikis:** bounded context for agents working with teammates. These should contain only what that role/team/project needs.
- **Hot memory (`MEMORY.md` / `USER.md`):** tiny every-turn steering, not an archive.

Correctness comes from routing every fact to the narrowest useful layer. If a teammate's agent only needs finance context, give it the finance wiki, not the user's whole personal GBrain. If an agent needs a broad personal pattern, pull it from GBrain on demand instead of copying it into hot memory.

For multiplayer agent products, treat GBrain/scoped wikis as the **context and access layer**, not the whole product. The second half is the collaboration surface: where humans invite agents, see what context the agent can access, review its work, and hand off decisions without losing provenance.

## Security and privacy notes

- GBrain is local-first; it does not call external APIs.
- The SQLite database can contain sensitive notes if the user stores sensitive notes.
- Do not commit `gbrain.db`, WAL files, or profile memory files.
- Keep `.gitignore` rules for local state and databases.
- Treat the database as private runtime data, not source code.

Recommended ignore patterns:

```gitignore
.gbrain/
gbrain/
*.db
*.db-wal
*.db-shm
```

Use narrower ignore rules if the repository intentionally tracks other SQLite fixtures.

## Test checklist

For a bundled Hermes provider, verify:

```bash
python -m pytest tests/plugins/test_gbrain_plugin.py -q
```

Minimum coverage:

- extraction for URLs, paths, handles, tags, quotes, capitalized phrases, aliases
- deduplication of repeated entities
- SQLite note insertion and search
- FTS5 search plus LIKE fallback behavior
- entity linking by note and by entity
- provider initialization under the active Hermes home
- `gbrain_note` actions: `add`, `search`, `links`, `stats`
- `on_memory_write` mirroring for add, replace, and remove
- path/query escaping so raw file paths and user text do not break search

## Recommended release path

1. Keep the code in `plugins/memory/gbrain/` while it depends on Hermes internals.
2. Test it as a bundled memory provider.
3. Publish the operator guide here.
4. Upstream it into Hermes Agent if it is broadly useful.
5. Only split it into a standalone repo if Hermes plugin distribution supports clean external installation.

The sane default: **code in Hermes Agent, guide in hermes-guide**. Anything else is a tiny architecture tax with a fake mustache.
