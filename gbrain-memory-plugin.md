# GBrain and the Legacy Hermes Memory Plugin

> **Status: historical integration notes, not a bundled-provider setup guide.** The repository previously linked here as `majesticlabs-dev/hermes-gbrain-plugin` is archived. At revision `83fee98a48f58639549c0d7b5c37ed781c805a8e`, its README calls the implementation **Majestic Brain**, with provider `majestic-brain` and tool `majestic_brain_note`, not `gbrain` / `gbrain_note`.

Current upstream Hermes does not contain the previously documented `plugins/memory/gbrain/`, `scripts/gbrain_backfill.py`, or `tests/plugins/test_gbrain_plugin.py`. Do not run those paths or set `memory.provider: gbrain` on a fresh install based on this page.

## Distinguish the integrations

These are separate projects and storage contracts:

1. **Legacy standalone note provider:** [archived source](https://github.com/majesticlabs-dev/hermes-gbrain-plugin/tree/83fee98a48f58639549c0d7b5c37ed781c805a8e). Its README describes local SQLite/FTS5 notes, deterministic extraction, graph-style links, prefetch recall, and mirroring of explicit memory writes. The package is `majestic_brain/`. Its old name does not make it an upstream bundled provider or prove compatibility with current Hermes.
2. **GBrain CLI / retrieval deployment:** [garrytan/gbrain](https://github.com/garrytan/gbrain). Commands such as `gbrain sources list`, `gbrain put`, and `gbrain get` belong to that independently installed system, not the legacy plugin's note tool. Discover its actual source, configuration, storage, and network behavior before relying on it.
3. **Other community GBrain plugins:** Current Hermes' plugin catalog contains independently maintained integrations. A matching product name does not make their provider IDs, tools, migration scripts, or access boundaries interchangeable.

The archived source's README also contains a malformed installation URL. This Guide intentionally does not reproduce it or invent a replacement supported install procedure. Review the selected plugin's manifest, current Hermes compatibility, and migration behavior before installation.

## Useful legacy design

The standalone implementation used explicit notes with entity extraction for URLs, file paths, handles, tags, quoted phrases, capitalized names, and aliases. Shared entities connected related notes; FTS5 search could fall back to SQLite `LIKE`.

That design is useful for small inspectable note stores. It is not a secret store, automatic transcript archive, or proof that a separate GBrain deployment never calls external services. The legacy provider's no-network claim applies to that implementation only.

## Current Hermes memory setup

Start with [official memory-provider documentation](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) and the chosen plugin's source. Safe discovery commands include:

```bash
hermes memory status
hermes plugins list
```

After an explicitly approved installation, `hermes memory setup` can select a provider that is actually available. Do not assume a provider will appear because its name occurs in a community guide. Restart the relevant session after a provider change and verify recall and writes against that provider's real interface.

## Memory routing pattern

- Keep `MEMORY.md` / `USER.md` compact for every-session steering.
- Keep authored knowledge, project decisions, and work state in their canonical documents or task system.
- Use a configured retrieval layer for durable information that does not belong in the hot cache.
- A profile-specific page such as `hermes/profile-memory-overflow/<profile>` is an operator convention for a separately configured GBrain corpus, not an upstream Hermes requirement.
- Verify the durable destination by reading it back before pruning the original. Do not delete memory merely because a migration command exited zero.

## Privacy and migration checks

Determine which system owns the underlying data, whether profiles share it, which APIs it contacts, and how backups work. Keep notes, databases, WAL/SHM files, and raw sessions out of public repositories. Test migration on a copy first; do not apply the legacy SQLite schema or missing backfill scripts to a different GBrain backend.

This page preserves the integration history and design tradeoffs. It does not certify a current deployment or authorize installing or migrating one.
