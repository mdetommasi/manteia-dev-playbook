# Coding Standards — Python

- **Tooling**: `ruff` (lint), `black` (format), `mypy` (types), `pytest` + `coverage`.
- **Struttura**: `src/<package>/...`, `tests/` mirror dei moduli, `pyproject.toml` centralizza config.
- **Error Handling**: eccezioni specifiche, no `except: pass`, log strutturati.
- **Config**: `pydantic` o `dynaconf`; disaccoppiare da env (12-factor).
- **I/O**: funzioni pure dove possibile, side-effects confinati.
- **Prompt & Templates**: versionare i prompt come file (`.prompt.md`) con variabili esplicite.

# Documentazione automatica con MkDocs, docstring (Google) e GitHub Actions

Questa sezione spiega **come generare e pubblicare automaticamente la documentazione** partendo dalle **docstring in stile Google** del codice Python, con:
- **MkDocs + Material** e **mkdocstrings** (estrazione API dal codice)
- **GitHub Actions** (build & deploy su GitHub Pages)
- **VS Code**: generazione **automatica** di docstring dalla firma (plugin) e **snippet** personalizzati

---

## Requisiti e pacchetti
Nel tuo progetto Python (con sorgenti in `src/<package>`), aggiungi le dipendenze per la doc:

```bash
pip install mkdocs mkdocs-material mkdocstrings[python] mkdocs-autorefs mkdocs-gen-files
```

- **mkdocstrings[python]**: estrae le API dalle docstring e le rende in pagine Markdown/HTML.
- **mkdocs-autorefs**: cross-reference automatico tra pagine/sezioni.
- **mkdocs-gen-files**: permette di **generare** file Markdown (es. indice API) al volo in fase di build.

> Opzionale: **pydocstyle** o **ruff** (regole `D***`) per linting delle docstring; **interrogate** per copertura docstring.

---

## Configurazione di MkDocs (`mkdocs.yml`)
Esempio di configurazione orientata a **Google style** e a una sezione **Reference** generata da `mkdocstrings`:

```yaml
site_name: Manteia AI Playbook
theme:
  name: material
docs_dir: docs
use_directory_urls: true

plugins:
  - search
  - autorefs
  - mkdocstrings:
      default_handler: python
      handlers:
        python:
          options:
            docstring_style: google
            show_root_heading: true
            show_source: false
            separate_signature: true
            merge_init_into_class: true
            heading_level: 2
            show_if_no_docstring: false

nav:
  - Home: index.md
  - Reference:
      - API Index: reference/index.md       # generato (vedi sezione successiva)
```

> Se preferisci non generare l’indice via script, puoi creare pagine manualmente e inserire blocchi `mkdocstrings`, es.:  
> `reference/my_module.md` con:  
> ```md
> # my_module
> ::: <package>.my_module
> ```

---

## Generare automaticamente la sezione Reference
Con **mkdocs-gen-files** puoi creare dinamicamente le pagine Reference in base ai moduli presenti in `src/<package>`. Aggiungi uno script, ad esempio **`docs/gen_ref_pages.py`**:

```python
# docs/gen_ref_pages.py
from pathlib import Path
import mkdocs_gen_files

package = "<package>"  # <-- sostituisci con il tuo package
root = Path("src") / package
nav = mkdocs_gen_files.Nav()

for path in sorted(root.rglob("*.py")):
    # ignora __init__.py se vuoi
    mod_path = path.relative_to("src").with_suffix("")
    mod_name = ".".join(mod_path.parts)
    doc_path = Path("reference", *mod_path.parts).with_suffix(".md")
    nav_path = list(mod_path.parts)

    with mkdocs_gen_files.open(doc_path, "w") as fd:
        fd.write(f"# {mod_name}\n\n")
        fd.write(f"::: {mod_name}\n")

    mkdocs_gen_files.set_edit_path(doc_path, path)
    nav[tuple(nav_path)] = doc_path

# crea un indice per Reference
with mkdocs_gen_files.open("reference/index.md", "w") as fd:
    fd.write("# API Reference\n\n")
    fd.writelines(nav.build_literate_nav())
```

E poi **abilita il hook** in `mkdocs.yml`:
```yaml
hooks:
  - docs/gen_ref_pages.py
```

> In alternativa, se non vuoi usare hook, lancia `python docs/gen_ref_pages.py` in un pre-step per generare i file sotto `docs/reference/` prima di `mkdocs build`.

---

## Docstring in stile Google (guida rapida)

Esempio per una funzione e una classe:

```python
def compute_ic(voltage: list[float], current: list[float], dt: float) -> list[float]:
    """Calcola l'Incremental Capacity (IC) a partire da V, I e dt.

    Args:
        voltage: Serie di valori di tensione (V).
        current: Serie di valori di corrente (A), segno secondo convenzione.
        dt: Passo temporale in secondi.

    Returns:
        Lista di valori IC (A/V) con stessa lunghezza degli ingressi.

    Raises:
        ValueError: Se le liste hanno lunghezze diverse o dt <= 0.
    """
    ...

class SoHEstimator:
    """Stima lo State of Health (SoH) da feature temporali.

    Attributes:
        window_size: Dimensione della finestra di analisi.
        method: Metodo di stima ("linear", "xgb", ...).
    """

    def __init__(self, window_size: int = 256, method: str = "linear") -> None:
        """Inizializza l'estimatore.

        Args:
            window_size: Dimensione finestra per feature engineering.
            method: Metodo di regressione usato per la stima.
        """
        ...
```

**Suggerimenti di stile:**
- Usa **`Args/Returns/Raises/Attributes/Examples`**.
- Mantieni le descrizioni **brevi e chiare**, aggiungi esempi minimi.
- Documenta **parametri e tipi** anche se usi type hints (aiuta il rendering).

---

## VS Code — docstring automatiche e snippet

### Plugin consigliato
- **autoDocstring – Python Docstring Generator**: genera docstring **Google**/NumPy/PEP257 dalla **firma** (type hints). Una volta installato:
  - Imposta formato **Google** in *Settings* → `autoDocstring.docstringFormat: "google"`.
  - In un file `.py`, posizionati sulla funzione, scrivi `"""` e conferma il suggerimento del plugin.

### Snippet personalizzati
Crea `.vscode/python.code-snippets` nel repo per snippet veloci, ad esempio:

```json
{
  "Docstring Google (funzione)": {
    "prefix": "gdoc",
    "body": [
      """"${1:Descrizione breve}.",
      "",
      "Args:",
      "    ${2:param}: ${3:descrizione}.",
      "",
      "Returns:",
      "    ${4:tipo}: ${5:descrizione}.",
      "",
      "Raises:",
      "    ${6:Eccezione}: ${7:quando}.",
      """""
    ],
    "description": "Template Google docstring"
  }
}
```

> Così puoi scrivere `gdoc` + `TAB` per inserire un template Google e completarlo rapidamente.

---

## Integrazione in GitHub Actions (build & deploy)

Se stai già usando **GitHub Pages** (soluzione B del nostro playbook), basta **aggiungere i pacchetti** necessari e riutilizzare il workflow. Esempio completo:

```yaml
name: deploy-docs
on:
  push:
    branches: [ main ]

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - name: Install doc tools
        run: |
          python -m pip install -U pip
          pip install mkdocs mkdocs-material mkdocstrings[python] mkdocs-autorefs mkdocs-gen-files
      - name: Build (MkDocs)
        run: mkdocs build -d site
      - uses: actions/upload-pages-artifact@v3
        with: { path: './site' }

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

> Se vuoi **caching pip**, aggiungi un `requirements.txt` con le dipendenze di doc e configura `cache-dependency-path` in `setup-python`.

---

## Qualità e policy docstring (opzionale ma consigliato)

### Ruff (regole pydocstyle)
Nel `pyproject.toml`:
```toml
[tool.ruff]
extend-select = ["D"]                 # attiva regole docstring (pydocstyle)
ignore = ["D203", "D213"]             # preferisci D211/D212 (stile Google)
docstring-convention = "google"
```

### pydocstyle (alternativa o insieme a ruff)
In `pyproject.toml` o `pydocstyle.ini`:
```toml
[tool.pydocstyle]
convention = "google"
add-ignore = "D105"   # opzionale: ignora doc per magic methods, ecc.
```

Aggiungi il check a **pre-commit**:
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.8
    hooks:
      - id: ruff
        args: [--fix]
  - repo: https://github.com/PyCQA/pydocstyle
    rev: 6.3.0
    hooks:
      - id: pydocstyle
```

---

## Workflow di lavoro suggerito
1. **Scrivi il codice con type hints** e genera docstring con **autoDocstring** o **snippet VS Code**.
2. **Lint delle docstring** con Ruff/pydocstyle (pre-commit e CI).
3. **Build locale della doc** con `mkdocs serve` per vedere il rendering delle API.
4. **Push su `main`**: GitHub Actions builda e **pubblica** su GitHub Pages.
5. Le pagine **Reference** si aggiornano automaticamente appena committi nuove docstring.

---

## Troubleshooting
- **La pagina Reference è vuota:** verifica che il package sia in `src/<package>` e che i moduli abbiano docstring; controlla lo script `gen_ref_pages.py` e il valore di `<package>`.
- **Link rotti tra pagine:** aggiungi `mkdocs-autorefs` e usa titoli/coincidenze coerenti; valida la `nav` nel `mkdocs.yml`.
- **Docstring non in Google style:** verifica le impostazioni del plugin VS Code e/o le regole di Ruff/pydocstyle.
- **Build Actions fallisce:** assicurati di avere `mkdocs.yml` in root, `docs/index.md` presente, dipendenze installate e **checkout** eseguito prima della build.

---

## Esempio rapido end-to-end (comandi)
```bash
# 1) Installa tool
pip install mkdocs mkdocs-material mkdocstrings[python] mkdocs-autorefs mkdocs-gen-files

# 2) Prova locale
mkdocs serve

# 3) Commit & push (GitHub Actions pubblica su Pages)
git add .
git commit -m "docs: auto API reference via mkdocstrings"
git push origin main
```

---

**Pronto per il playbook**: inserisci questo file come `docs/practices/docs-automation.md` e aggiungi la voce in `mkdocs.yml`. In caso di dubbi, prova prima in locale con `mkdocs serve`.
