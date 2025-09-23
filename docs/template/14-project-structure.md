# Project Template

Struttura di esempio: moduli chiari, YAML configs, CLI Typer, test.

[Scarica il pacchetto](../downloads/manteia_project_template.zip)


## Layout
~~~text

<project_root>/
├─ README.md
├─ pyproject.toml
├─ .pre-commit-config.yaml
├─ .gitignore
├─ Makefile
├─ configs/
│ ├─ default.yaml
│ ├─ data/
│ ├─ model/
│ └─ train/{pretrain.yaml,fine_tune.yaml}
├─ src/<package>/
│ ├─ init.py
│ ├─ cli.py
│ ├─ utils/{config.py,logging.py,paths.py}
│ ├─ data/{datasets.py}
│ ├─ models/
│ ├─ training/{lit_module.py,trainer.py,callbacks.py}
│ ├─ eval/{metrics.py}
│ └─ plots/
├─ scripts/{run_train.sh,download_data.sh}
├─ notebooks/
│ ├─ etl/.gitkeep
│ └─ model/.gitkeep
├─ tests/<package>
│ ├─ test_datasets.py
│ ├─ test_models_shape.py
│ └─ test_cli.py
└─ .github/workflows/ci.yml
~~~

## CLI standard
- `train`, `finetune`, `reconstruct`, `eval`, `plot`, `split`.
- Config a riga di comando: `--config configs/train/pretrain.yaml`.