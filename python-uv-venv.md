# Python Environment with uv + venv

Hermes should run from a project virtualenv when you are developing from a source checkout or maintaining a local operator installation.

If `hermes doctor` shows this:

```text
◆ Python Environment
  ✓ Python 3.14.2
  ⚠ Not in virtual environment (recommended)
```

then the `hermes` command is resolving to a global console script, not the project venv.

## Recommended setup

From the Hermes checkout:

```bash
cd ~/.hermes/hermes-agent
```

Use the repo's existing environment name when it already exists. In this checkout the environment is `venv`:

```bash
uv venv venv --python 3.11
uv pip install --python venv/bin/python -e ".[all,dev]"
```

If `venv` already exists and uses the right Python version, do not recreate it. Just install into it:

```bash
uv pip install --python venv/bin/python -e ".[all,dev]"
```

## Verify the venv directly

```bash
venv/bin/python - <<'PY'
import sys
print(sys.executable)
print(sys.version.split()[0])
print(sys.prefix != sys.base_prefix)
PY

venv/bin/hermes --version
venv/bin/hermes doctor
```

Expected doctor result:

```text
◆ Python Environment
  ✓ Python 3.11.x
  ✓ Virtual environment active
```

## Make `hermes` resolve to the venv

A working venv is not enough. Your shell may still find a global `hermes` first, especially from mise/asdf/pyenv.

Check resolution:

```bash
type -a hermes
command -v hermes
head -1 "$(command -v hermes)"
```

If `hermes` points at a global Python install, add a small wrapper in a PATH directory that wins for your shell. Example:

```bash
#!/usr/bin/env bash
unset PYTHONPATH
unset PYTHONHOME
exec "$HOME/.hermes/hermes-agent/venv/bin/hermes" "$@"
```

Install it somewhere early on PATH, for example:

```bash
sudo install -m 755 hermes /opt/homebrew/bin/hermes
```

If `/opt/homebrew/bin` is user-writable on your machine, `sudo` is not required.

## Clear shell command cache

Some shells cache command paths. After changing wrappers or PATH:

```bash
rehash 2>/dev/null || true
hash -r 2>/dev/null || true
command -v hermes
hermes doctor
```

If it still reports the wrong Python, open a fresh terminal tab and re-run `command -v hermes`.

## Final checklist

Before calling the environment fixed, verify all of these:

- `venv/bin/python` is Python 3.11+.
- `venv/bin/hermes doctor` says `Virtual environment active`.
- `command -v hermes` points to the intended wrapper or venv entry point.
- Plain `hermes doctor` in a fresh shell also says `Virtual environment active`.
- `uv pip check --python venv/bin/python` reports compatible packages.
