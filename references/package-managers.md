# Package Manager Reference

Commands, manifest field mappings, and install instructions for each supported package manager.

---

## Pixi

### Detect
- `pixi.toml` exists at the project root

### Query installed packages
```bash
# Default environment
pixi list --json

# Named environment
pixi list --json --environment <env-name>
```

Output shape (array of objects):
```json
[
  { "name": "numpy", "version": "1.26.4", "build": "py313h...", "channel": "conda-forge", "kind": "conda" },
  { "name": "requests", "version": "2.31.0", "kind": "pypi" }
]
```

Normalize `kind` to `conda` or `pypi` — this determines the purl type.

### Manifest fields (pixi.toml)
```toml
[dependencies]
numpy = ">=1.26"
python = "3.13.*"

[pypi-dependencies]
requests = ">=2.31"

[feature.dev.dependencies]
pytest = ">=8"
ruff = ">=0.4"
```

- `[dependencies]` → conda packages (purl type: `conda`)
- `[pypi-dependencies]` → PyPI packages (purl type: `pypi`)
- `[feature.*.dependencies]` → dev/optional, include when `INCLUDE_DEV=true`

### Enumerate all environments
Read the `[environments]` table from `pixi.toml`. Each key is an environment name.
If the table is absent, the only environment is `default`.

```bash
# List installed environments and their solve groups
pixi info --json 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); print('\n'.join(d.get('environments_info', {}).keys()))"
```

Alternatively, parse `pixi.toml` directly with a regex or TOML parser to extract keys under `[environments]`.

### Install cyclonedx globally (do NOT install into the project pixi environment)
```bash
# Install globally — adds cyclonedx-py to ~/.pixi/bin/
pixi global install cyclonedx-bom

# Check if already installed
pixi global list 2>/dev/null | grep -i cyclonedx

# The global env's Python (has cyclonedx-python-lib available):
~/.pixi/envs/cyclonedx-bom/bin/python
```

---

## uv

### Detect
- `uv.lock` exists, OR
- `pyproject.toml` contains `[tool.uv]`

### Query installed packages
```bash
# Inside an active uv venv
uv pip list --format json

# Alternatively, export to requirements format
uv export --no-hashes --format requirements-txt
```

Output shape (JSON array):
```json
[
  { "name": "requests", "version": "2.31.0" },
  { "name": "certifi", "version": "2024.2.2" }
]
```

### Manifest fields (pyproject.toml with uv)
```toml
[project]
dependencies = [
    "requests>=2.31",
    "httpx[http2]>=0.27",
]

[project.optional-dependencies]
dev = ["pytest>=8", "ruff>=0.4"]

[tool.uv]
dev-dependencies = ["pytest>=8"]
```

- `[project.dependencies]` → direct runtime deps
- `[project.optional-dependencies.dev]` and `[tool.uv.dev-dependencies]` → dev deps (include when `INCLUDE_DEV=true`)

### Install cyclonedx-python-lib
```bash
uv add --dev cyclonedx-python-lib
# or for one-time use:
uv tool install cyclonedx-bom
```

---

## conda

### Detect
- `environment.yml` or `conda.yml` exists, OR
- `conda` binary is on PATH and `pixi.toml` is absent

### Enumerate all environments
```bash
conda info --json
```

Output shape (relevant fields):
```json
{
  "envs": [
    "/opt/conda",
    "/opt/conda/envs/myenv",
    "/opt/conda/envs/data-science"
  ],
  "active_prefix": "/opt/conda/envs/myenv"
}
```

Derive environment name from path:
- First entry (the root) → name `base`
- All others → `os.path.basename(path)`

### Query installed packages for a specific environment
```bash
# By prefix path (works without activation — preferred for multi-env)
conda list --prefix /opt/conda/envs/myenv --json

# By name (requires the env to be installed)
conda list --name myenv --json
```

Prefer `--prefix` so environments can be queried without activation. Obtain the prefix path from `conda info --json`.

Output shape (array of objects):
```json
[
  { "name": "numpy", "version": "1.26.4", "build_string": "py313h...", "channel": "conda-forge" },
  { "name": "pip", "version": "24.0", "channel": "defaults" }
]
```

Packages installed via pip inside conda show `channel: "pypi"`.

### Manifest fields (environment.yml)
```yaml
name: myenv
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.13
  - numpy>=1.26
  - pip:
    - requests>=2.31
    - httpx
```

- Top-level list items → conda packages (`purl type: conda`)
- Items under `pip:` → PyPI packages (`purl type: pypi`)

### Install cyclonedx globally (do NOT install inside a conda project env)
```bash
# Preferred: install via pixi global (if pixi is available)
pixi global install cyclonedx-bom

# Fallback: install into the system/user Python, not inside the project env
pip install --user cyclonedx-python-lib
```

---

## pip

### Detect
- `requirements.txt` or `requirements-*.txt` exists, OR
- `setup.py` or `setup.cfg` exists, OR
- `pyproject.toml` without `[tool.uv]` or `[tool.pixi]`

### Query installed packages
```bash
pip list --format json
```

Output shape:
```json
[
  { "name": "requests", "version": "2.31.0" },
  { "name": "certifi", "version": "2024.2.2" }
]
```

### Manifest fields

**requirements.txt** — one package per line, PEP 508 format:
```
requests>=2.31
httpx[http2]>=0.27
# -r other-requirements.txt  ← follow recursive includes
```

**pyproject.toml** (no uv):
```toml
[project]
dependencies = ["requests>=2.31"]

[project.optional-dependencies]
dev = ["pytest>=8"]
```

**setup.cfg**:
```ini
[options]
install_requires =
    requests>=2.31
    httpx>=0.27
```

### cyclonedx-py CLI (pip/uv only)

Install:
```bash
pip install cyclonedx-bom
```

Usage:
```bash
# From active environment
cyclonedx-py environment --output-format JSON --outfile sbom.cyclonedx.json --schema-version 1.4

# From requirements.txt (no environment needed)
cyclonedx-py requirements -r requirements.txt --output-format JSON --outfile sbom.cyclonedx.json --schema-version 1.4
```

### Install cyclonedx-python-lib
```bash
pip install cyclonedx-python-lib
```

---

## Package URL (purl) formats by ecosystem

| Ecosystem | purl format | Example |
|---|---|---|
| PyPI | `pkg:pypi/<name>@<version>` | `pkg:pypi/requests@2.31.0` |
| conda (conda-forge) | `pkg:conda/conda-forge/<name>@<version>` | `pkg:conda/conda-forge/numpy@1.26.4` |
| conda (defaults) | `pkg:conda/defaults/<name>@<version>` | `pkg:conda/defaults/python@3.13.0` |
| conda (unknown channel) | `pkg:conda/<name>@<version>` | `pkg:conda/scipy@1.13.0` |

Use the `channel` field from the package list output to fill the namespace. If channel is `pypi` or absent in a conda list, use `pkg:pypi`.

---

## Name normalization (PEP 503)

Canonical form: lowercase, replace `_` and `.` with `-`.

```python
import re
def canonical(name: str) -> str:
    return re.sub(r"[-_.]+", "-", name).lower()
```

Use this when matching manifest names against installed package names.
