---
name: python
description: "Python conventions built entirely on uv. Use whenever installing Python packages, creating or activating a virtual environment, installing or running a Python CLI tool, choosing or pinning an interpreter version, running a Python script, setting up a new Python project, or adding dependencies to an existing one. Also use when about to write any pip, pipx, pyenv, poetry, virtualenv, or `python -m venv` command — those are all replaced here."
---

# Python

`uv` is the only entry point for Python work: packages, tools, virtual environments, and interpreters.

**The rule that matters: nothing is ever installed into a system or user-level `site-packages`.** Every package lives in either a project venv or a uv-managed tool venv. A system Python that has been written to is contaminated and hard to clean up, so treat the boundary as absolute rather than as a default to be overridden when convenient.

Never invoke `pip`, `pipx`, `poetry`, `pyenv`, `virtualenv`, or `python -m venv` directly.

## Replacements

| Instead of | Use |
| --- | --- |
| `python3 -m venv .venv`, `virtualenv` | `uv venv` |
| `pip install <pkg>` | `uv add <pkg>` (project) or `uv pip install <pkg>` (bare venv) |
| `pip install -r requirements.txt` | `uv pip install -r requirements.txt` |
| `pip uninstall <pkg>` | `uv remove <pkg>` or `uv pip uninstall <pkg>` |
| `pip freeze`, `pip list` | `uv pip freeze`, `uv pip list` |
| `pipx install <tool>` | `uv tool install <tool>` |
| `pipx run <tool>` | `uvx <tool>` |
| `pyenv install 3.13` | `uv python install 3.13` |
| `pyenv local 3.13` | `uv python pin 3.13` |
| `python script.py` | `uv run script.py` |
| `source .venv/bin/activate` | not needed — `uv run` and `uv pip` auto-detect `./.venv` |

## Projects (preferred)

Anything with — or deserving — a `pyproject.toml`. uv creates and manages `.venv` for you.

```bash
uv init                 # new project
uv add <package>        # dependency (updates pyproject.toml + uv.lock)
uv add --dev <package>  # dev-only dependency
uv remove <package>
uv sync                 # reconcile .venv with uv.lock
uv run <command>        # run inside the project env
```

Commit `pyproject.toml` and `uv.lock`. Never commit `.venv/`.

Prefer this over a bare venv even for small things — the lockfile makes the environment reproducible, and `uv run` removes any question of which interpreter is active.

## Bare virtual environments

Only when a `pyproject.toml` is not appropriate, such as an existing requirements.txt repo you don't own:

```bash
uv venv                            # creates ./.venv
uv pip install <package>           # installs into ./.venv, no activation needed
uv pip install -r requirements.txt
```

`uv pip` resolves the target environment from `./.venv` or `$VIRTUAL_ENV`. If neither exists it errors rather than falling back to system Python — that failure is correct behavior, so create a venv instead of reaching for `--system`.

Watch for an inherited `VIRTUAL_ENV` from an unrelated directory; it wins over `./.venv` and will silently install into the wrong environment. Check it before installing if the shell state is uncertain.

## One-off scripts and tools

Never install something permanently just to run it once.

```bash
uvx ruff check .                 # run a CLI tool in a throwaway env
uv tool install ruff             # install a CLI tool persistently, in its own venv
uv run --with httpx script.py    # ad-hoc dep; does not touch the project env
uv run --no-project script.py    # run ignoring any surrounding project
```

`uv tool install` gives each tool its own isolated venv and links only the executable onto `PATH`, so tools never share a dependency graph with each other or with a project.

For standalone scripts, prefer PEP 723 inline metadata so dependencies travel with the file:

```python
# /// script
# requires-python = ">=3.12"
# dependencies = ["httpx"]
# ///
```

Then `uv run script.py` builds the environment on demand — no project, no venv to manage.

## Interpreters

Prefer uv-managed standalone builds over whatever Python the OS ships:

```bash
uv python install 3.13   # download a managed interpreter
uv python pin 3.13       # write .python-version for this project
uv python list
```

Set `UV_MANAGED_PYTHON=1` (or pass `--managed-python`) to make uv refuse to fall back to a system interpreter.

## Never

- `sudo pip`, `sudo uv`, or any install as root
- `pip install --user`, `pip install --break-system-packages`
- `uv pip install --system` — installs into the system interpreter and defeats the point
- Modifying the Homebrew or OS-vendored Python in any way
- Committing `.venv/`
