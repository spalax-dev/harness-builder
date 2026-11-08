# Python stack reference

Covers `uv`, Poetry and pip/venv projects. Prefer project-owned commands
(`uv run`, `poetry run`, `make`, `just`) over global binaries.

## Detection

| Signal                                     | Meaning                                         |
| ------------------------------------------ | ----------------------------------------------- |
| `pyproject.toml`                           | Modern project; inspect `[project]`, `[tool.*]` |
| `uv.lock`                                  | uv                                              |
| `poetry.lock`                              | Poetry                                          |
| `requirements.txt`                         | pip/venv                                        |
| `ruff.toml` / `[tool.ruff]`                | Ruff                                            |
| `mypy.ini` / `[tool.mypy]`                 | mypy                                            |
| `[tool.pyright]` / `pyrightconfig.json`    | pyright                                         |
| `pytest.ini` / `[tool.pytest.ini_options]` | pytest                                          |
| `src/<pkg>/`                               | src layout                                      |
| `.venv/`                                   | virtualenv                                      |

## Commands by gate

| Gate        | uv                                   | Poetry                              | pip/venv                           |
| ----------- | ------------------------------------ | ----------------------------------- | ---------------------------------- |
| format      | `uv run ruff format --check .`       | `poetry run ruff format --check .`  | `.venv/bin/ruff format --check .`  |
| lint        | `uv run ruff check .`                | `poetry run ruff check .`           | `.venv/bin/ruff check .`           |
| type        | `uv run mypy .` or `uv run pyright`  | `poetry run mypy .`                 | `.venv/bin/mypy .`                 |
| build       | `uv build`                           | `poetry build`                      | `python -m build`                  |
| test full   | `uv run pytest -q`                   | `poetry run pytest -q`              | `.venv/bin/pytest -q`              |
| test target | `uv run pytest tests/test_x.py`      | `poetry run pytest tests/test_x.py` | `.venv/bin/pytest tests/test_x.py` |
| contract    | import the package / inspect wheel   | same                                | same                               |
| structure   | `src/<pkg>` layout, no `__pycache__` | same                                | same                               |
| docs        | mkdocs/pdoc build or docs word count | same                                | same                               |

Formatter alternatives: Black (`black --check .`) and isort if the project uses
them. Linter alternatives: flake8, pylint. Type alternatives: pyright, basedpyright.
Use what the project already configures.

## Targeted spec discovery

```js
// CONFIG.specs.pattern
/^tests\/.*\.py$|_test\.py$|^test_.*\.py$/;

// CONFIG.specs.candidates
if (file.endsWith(".py")) {
  const dir = dirname(file);
  const name = basename(file, ".py");
  return [
    `${dir}/test_${name}.py`,
    `${dir}/${name}_test.py`,
    `tests/test_${name}.py`,
  ];
}
```

Run targeted tests with the same runner (`pytest`), not a new one.

## Structure examples

- Required: `src/<pkg>/`, `tests/`, `pyproject.toml`.
- Forbidden: imports from `tests/` into production, `__pycache__` committed,
  `sys.path` hacks.
- Max lines: optional; use the project’s own limit.
- `py.typed` if the package advertises types.

## Contract examples

- `python -c "import <pkg>"` succeeds from the installed package.
- Wheel contains the package and `py.typed` (if typed).
- CLI entry point (`[project.scripts]`) is importable.
- JSON schema validates an example resource.
- Docs build with `mkdocs build --strict` (if MkDocs is configured).

## Budgets

- Light: changed `ruff` format/lint + structure/docs — seconds.
- Medium: + `mypy`/`pyright` + targeted `pytest` — usually 5–30 s.
- Full: + build + full `pytest` + docs build — measure; suites vary widely.

## Pitfalls

- Do not use a global `python`/`pytest` if the project has a virtualenv; use
  `uv run`, `poetry run` or `.venv/bin/...`.
- Do not run `mypy` on `.venv/`, `build/`, `dist/`, `__pycache__/`.
- Do not treat `pytest` collection errors as test failures without surfacing
  the collection output; a missing dependency is not a failing assertion.
- Do not run `build` before the contract check unless the contract reads the
  wheel; if it does, run it after.
