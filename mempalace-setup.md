# MemPalace Setup for Hermes

Installed and configured 2026-04-14 for Hermes on macOS Apple Silicon.

## Installation

```bash
# Create dedicated venv (NOT in hermes-agent's venv)
python3 -m venv ~/mempalace-venv

# Activate and install
source ~/mempalace-venv/bin/activate
pip install mempalace

# Installed version: 3.2.0
```

No ARM64 segfault hit (Issue #74 is known but did not manifest on this install).
Hook system skipped per known issue (hooks broken on PyPI, use CLI/Python API directly).
Raw mode only — AAAK compression not used (it regresses recall: 84.2% vs 96.6% raw).

## Initialization

```bash
source ~/mempalace-venv/bin/activate

# Init Hermes data directory (non-interactive)
mempalace init --yes ~/.hermes

# Init knowledge base
mempalace init --yes ~/kb
```

## Data Ingested

Two wings created:

### Wing: .hermes
- Sessions (4502 drawers) — all conversation history
- Cache (2491 drawers) — models_dev_cache.json
- Atlas archive (1378 drawers) — project reference files
- Skills (73 drawers) — skill definitions
- Pastes, research, scripts, hooks, etc.

### Wing: kb
- Archive (1022 drawers)
- Documentation (507 drawers) — notes/
- Ops (249 drawers) — operational data
- Inbox (201 drawers)
- Self (93 drawers) — agent identity/methodology
- Templates, projects, manual, state, scripts

### Mining commands
```bash
# Mine Hermes project data (runs ~10-15 min for 4698 files)
mempalace mine ~/.hermes --mode projects

# Mine knowledge base (runs ~1-2 min for 491 files)
mempalace mine ~/kb --mode projects
```

Total: 10,735 drawers indexed at setup time.

## Querying

### CLI search
```bash
source ~/mempalace-venv/bin/activate

# Basic search
mempalace search "why did we switch to GraphQL"

# Search within a wing
mempalace search "auth decisions" --wing kb

# Search within a room
mempalace search "pricing discussion" --wing kb --room documentation
```

### Wake-up context (for AI agents)
```bash
# Get ~600-900 token context for system prompt injection
mempalace wake-up

# Wing-specific wake-up
mempalace wake-up --wing kb
```

### Python API
```python
import sys
sys.path.insert(0, os.path.expanduser('~/mempalace-venv/lib/python3.9/site-packages'))

from mempalace.searcher import search_memories
results = search_memories("hermes configuration", palace_path="~/.mempalace/palace")
for r in results:
    print(r)
```

### Status check
```bash
mempalace status
```

## Known Issues

| Issue | Status | Workaround |
|-------|--------|------------|
| macOS ARM64 segfault (#74) | Not hit on this install | Run `mempalace repair` if it occurs |
| Hooks broken on PyPI (#110) | Skipped | Use CLI or Python API directly |
| AAAK regression vs raw | Not used | Raw mode only (96.6% vs 84.2%) |
| Shell injection in hooks (#110) | N/A | Skipped hooks entirely |
| ChromaDB pinning (#100) | Monitoring | Keep venv isolated |

## Palace Location

- Default palace path: `~/.mempalace/palace/`
- Config: `~/.mempalace/config.json`
- ChromaDB SQLite: `~/.mempalace/palace/chroma.sqlite3` (~70MB+)
- Entity files: `~/.hermes/entities.json`, `~/kb/entities.json`
- Wing configs: `~/.hermes/mempalace.yaml`, `~/kb/mempalace.yaml`

## Useful Commands Reference

```bash
# Always activate first
source ~/mempalace-venv/bin/activate

mempalace status                    # Show what's filed
mempalace search "query"            # Search everything
mempalace wake-up                   # Get context for AI system prompt
mempalace mine <dir> --mode projects   # Ingest project files
mempalace mine <dir> --mode convos     # Ingest conversation exports
mempalace repair                    # Rebuild index if corrupted
mempalace mcp                       # Show MCP setup command
```

## Integration with Hermes

To use MemPalace from within Hermes sessions, the AI can call:

```bash
source ~/mempalace-venv/bin/activate && mempalace search "<query>"
```

Or use the Python API to programmatically search and inject results into context.

For MCP integration with Claude:
```bash
claude mcp add mempalace -- python -m mempalace.mcp_server
```
