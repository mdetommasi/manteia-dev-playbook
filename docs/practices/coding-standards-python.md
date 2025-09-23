# Coding Standards — Python

- **Tooling**: `ruff` (lint), `black` (format), `mypy` (types), `pytest` + `coverage`.
- **Struttura**: `src/<package>/...`, `tests/` mirror dei moduli, `pyproject.toml` centralizza config.
- **Error Handling**: eccezioni specifiche, no `except: pass`, log strutturati.
- **Config**: `pydantic` o `dynaconf`; disaccoppiare da env (12-factor).
- **I/O**: funzioni pure dove possibile, side-effects confinati.
- **Prompt & Templates**: versionare i prompt come file (`.prompt.md`) con variabili esplicite.
