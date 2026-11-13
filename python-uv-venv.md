# Python Environment: Current PM Workflow and Legacy uv Notes

Current first-party Hermes installations use the built-in package manager (`hermes pm`) to select Python and dependencies. The previous recipe on this page — creating a Python 3.11 `venv`, installing `.[all,dev]` with raw uv, and replacing a global command with a hardcoded `venv/bin/hermes` wrapper — is not the current supported setup.

The filename is retained for existing links. A conventional venv can still exist in a legacy installation; its presence alone does not prove which runtime is active.

## Installed Hermes

Use the installation's published `hermes` launcher rather than guessing a venv path:

```bash
type -a hermes
command -v hermes
hermes --version
hermes pm status
hermes pm doctor
hermes doctor
```

Compare the selected environment with the installation owner and current lock. Do not run raw pip/uv against a PM-owned environment or overwrite an OS-wide launcher to silence an old virtualenv warning. Repairs and updates are separate mutations and should be approved before running them.

Current first-party installations use Python 3.14. The broader `>=3.11,<3.15` metadata range permits older installations to reach the updater; it is not a promise of full current runtime support on every Python in that range.

## Source development

Use a clean Bash shell outside an existing venv. From the intended Hermes source checkout, the official developer recipe is:

```bash
export HERMES_HOME="$HOME/hermes-dev-data"
export HERMES_RUNTIME_DIR="$HERMES_HOME/tools"
source ./activate
```

Activation can provision tools and sync dependencies. It is a setup action, not a read-only diagnostic. Keep these overrides process-local and restore the same values when returning to this checkout. Do not point a development checkout at a production profile accidentally.

Activation defines a worktree-scoped `hermes` function and selects the managed interpreter/dependency tree. It does not promise a conventional `venv/` directory or a `Virtual environment active` doctor line. Leave with `deactivate`.

Verify actual selection after authorized setup:

```bash
python -c "import sys, pm; print(sys.executable); print(pm.__file__)"
hermes --version
hermes pm status
```

The interpreter should resolve to the PM store and `pm.__file__` to the intended checkout. A successful activation alone does not prove all required dependencies are available. Follow the official developer workflow for dev dependencies and tests instead of treating `dev` as an old pip extra.

## Sources

- [Package Management — Developer workflow](https://hermes-agent.nousresearch.com/docs/reference/package-management#developer-workflow)
- [Installation](https://hermes-agent.nousresearch.com/docs/getting-started/installation)
