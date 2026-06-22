# generate-python-sbom-skill

Generate a CycloneDX 1.4 Software Bill of Materials (SBOM) for any Python project. Detects and supports Pixi, uv, pip, conda, and Poetry. If the required package manager is not installed, bootstraps it to a temporary directory, does the work, then removes it. Never modifies project environments.

---

## Contents

- [What it does](#what-it-does)
- [Quick start](#quick-start)
- [Arguments](#arguments)
- [Package manager support](#package-manager-support)
- [Multi-environment support](#multi-environment-support)
- [Output](#output)
- [Generation strategies](#generation-strategies)
- [Bootstrapping missing tools](#bootstrapping-missing-tools)
- [Invoking from different tools](#invoking-from-different-tools)
- [Examples](#examples)
- [Validating the output](#validating-the-output)
- [File structure](#file-structure)
- [Troubleshooting](#troubleshooting)

---

## What it does

1. Detects the package manager from manifest files (`pixi.toml`, `pyproject.toml`, `requirements.txt`, `environment.yml`, `uv.lock`, `poetry.lock`) in the project root
2. If the required package manager binary is missing, installs it temporarily to `$(mktemp -d)` and removes it when done
3. Reads direct dependencies from all manifest files present
4. For Pixi and conda, enumerates every environment and generates one SBOM per environment; pip and uv always produce a single SBOM
5. Queries installed packages (direct + transitive) from each environment
6. Generates a CycloneDX 1.4 SBOM in JSON or XML, marking each component as `direct` or `transitive`
7. Cleans up any temporarily bootstrapped tools

CycloneDX tooling is installed globally (via `pixi global install` or `pip install --user`) so no project environment is ever modified. If global installation also fails, a stdlib-only fallback generates a valid SBOM using only Python's built-in modules.

---

## Quick start

The shortest possible invocation — auto-detects everything from the current directory:

```
/generate-python-sbom
```

Specify a project directory explicitly:

```
/generate-python-sbom --path /path/to/my-project
```

Generate XML instead of JSON:

```
/generate-python-sbom --path /path/to/my-project --format xml
```

Pixi project — generate for one environment only:

```
/generate-python-sbom --path /path/to/my-project --env py312
```

Include dev dependencies in the direct-dep list:

```
/generate-python-sbom --path /path/to/my-project --include-dev
```

---

## Arguments

```
/generate-python-sbom [--path <project-dir>] [--format json|xml] [--output <path>] [--env <name>] [--include-dev]
```

All arguments are optional. The skill can auto-detect everything from the current directory with no arguments.

| Argument | Default | Description |
|---|---|---|
| `--path <project-dir>` | Current working directory | Absolute or relative path to the Python project root. Use this to target a project that is not the current directory. |
| `--format json\|xml` | `json` | CycloneDX output format. Both formats conform to CycloneDX 1.4 schema. |
| `--output <path>` | `<project-root>/sbom.cyclonedx[.<env>].<format>` | Override the output file path. When multiple environments are generated the environment name is suffixed automatically (e.g., `sbom.cyclonedx.default.json`, `sbom.cyclonedx.py312.json`). Supplying `--output` with a multi-environment project sets the base path; the env suffix is still appended. |
| `--env <name>` | All environments | Pixi and conda only. Limit generation to this one named environment. If the environment name does not exist the skill stops and lists available environments. Ignored for pip and uv. |
| `--include-dev` | false | Include dev, test, and optional dependency groups in the direct-dependency list. Affects `[feature.*.dependencies]` in `pixi.toml`, `[project.optional-dependencies]` in `pyproject.toml`, `[tool.uv.dev-dependencies]`, and similar. Does not change which packages are scanned — only which ones are tagged `direct` vs. `transitive`. |

### Variable resolution

Before any step runs, the skill resolves these variables from the arguments:

```
PROJECT_ROOT  = absolute path of --path, or pwd
FORMAT        = "json" unless --format xml
OUTPUT_BASE   = --output value, or "${PROJECT_ROOT}/sbom.cyclonedx"
ENV_FILTER    = --env value, or "" (empty = all environments)
INCLUDE_DEV   = "true" if --include-dev present, else "false"
```

---

## Package manager support

The skill detects the package manager by inspecting manifest files in `PROJECT_ROOT`. Detection follows this priority order — the first match wins.

| Priority | Manager | Detection signal |
|---|---|---|
| 1 | **Pixi** | `pixi.toml` exists at project root |
| 2 | **uv** | `uv.lock` exists, OR `pyproject.toml` contains `[tool.uv]` |
| 3 | **conda** | `environment.yml` or `conda.yml` exists at project root, OR `conda` is on PATH and no `pixi.toml` is present |
| 4 | **pip** | `requirements.txt` / `requirements-*.txt` exists, OR `setup.py` / `setup.cfg` exists, OR `pyproject.toml` without uv/pixi config |

When multiple manifest files are present (e.g., a Pixi project that also has `pyproject.toml`), the highest-priority manager is used for environment querying, but **all present manifest files are read** in the direct-dependency extraction step to produce a complete direct-dep list.

### What is read from each manifest

**`pixi.toml`**

| Section | Included when |
|---|---|
| `[dependencies]` | Always (conda packages) |
| `[pypi-dependencies]` | Always |
| `[feature.*.dependencies]` | Only when `--include-dev` is set |

**`pyproject.toml`**

| Section | Included when |
|---|---|
| `[project.dependencies]` | Always |
| `[project.optional-dependencies.*]` | Only when `--include-dev` |
| `[tool.uv.dev-dependencies]` | Only when `--include-dev` |
| `[tool.pdm.dev-dependencies]` | Only when `--include-dev` |

**`requirements.txt` / `requirements-*.txt` / `constraints.txt`**

Every non-comment, non-empty line that does not start with `-r` (recursive include) is treated as a direct dependency. Inline version specifiers (`requests>=2.31`) are accepted — only the package name is extracted.

**`environment.yml` / `conda.yml`**

The top-level `dependencies:` list items are conda packages. Items under the `pip:` sub-key are PyPI packages.

**`setup.py` / `setup.cfg`**

`install_requires` is read only when no `pyproject.toml` is present.

---

## Multi-environment support

### Pixi

The skill reads the `[environments]` table from `pixi.toml`. Each key in that table becomes a separate SBOM. If the table is absent, the only environment is `default`.

```toml
# pixi.toml — this produces three SBOMs:
# sbom.cyclonedx.default.json
# sbom.cyclonedx.py312.json
# sbom.cyclonedx.py313.json
[environments]
default = { ... }
py312   = { features = ["py312"], ... }
py313   = { features = ["py313"], ... }
```

To generate for one environment only, pass `--env py312`.

### conda

The skill scans `PROJECT_ROOT` (recursively) for all conda environment YAML files — `environment.yml`, `environment-*.yml`, `conda.yml`, and `conda-*.yml`. It generates one SBOM per file found.

Unlike Pixi (which queries pre-existing installed environments), **conda SBOMs are generated from a freshly created temporary environment**: the skill runs `conda env create --file <yaml_path> --prefix <TEMP_DIR>/conda-envs/<name>`, queries the fully resolved and installed package list with `conda list --prefix <prefix> --json`, generates the SBOM, then deletes the prefix directory. Using `--prefix` (not `--name`) keeps the environment entirely inside `TEMP_DIR` and never adds an entry to the user's conda environment registry. This gives a complete view of direct + transitive deps at their resolved conda-forge versions.

**Important:** creating the temporary environment requires downloading packages from conda-forge. This can take several minutes depending on the number of packages and network speed. The environment is removed immediately after the SBOM is written.

Example project layout:
```
my-project/
  environment.yml          → name: my-project   → sbom.cyclonedx.my-project.json
  envs/
    environment-gpu.yml    → name: my-project-gpu → sbom.cyclonedx.my-project-gpu.json
    environment-cpu.yml    → name: my-project-cpu → sbom.cyclonedx.my-project-cpu.json
```

If there is only one environment YAML in the project, no env-name suffix is added: `sbom.cyclonedx.json`.

To generate for one environment only, pass `--env <name>` where `<name>` matches the `name:` field in the target YAML file.

### pip / uv

Always produce a single SBOM named `sbom.cyclonedx.json` (or `.xml`). The `--env` argument is not applicable and is ignored with a warning if supplied.

**pip venv resolution** — the skill looks for an active virtualenv in the project directory in this order:

1. `<PROJECT_ROOT>/.venv/bin/pip`
2. `<PROJECT_ROOT>/venv/bin/pip`
3. `pip` on PATH

---

## Output

### Filename convention

| Scenario | Output filename |
|---|---|
| Single environment (pip / uv / pixi default-only) | `sbom.cyclonedx.json` |
| Multiple environments (pixi / conda) | `sbom.cyclonedx.<env-name>.json` |
| `--format xml` with multiple envs | `sbom.cyclonedx.<env-name>.xml` |
| `--output /path/to/report.json` (single env) | `/path/to/report.json` |
| `--output /path/to/report` (multiple envs) | `/path/to/report.<env-name>.json` |

### CycloneDX 1.4 JSON structure

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "serialNumber": "urn:uuid:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "version": 1,
  "metadata": {
    "timestamp": "2025-01-15T12:00:00+00:00",
    "tools": [
      { "vendor": "generate-python-sbom", "name": "generate-python-sbom", "version": "1.0.0" }
    ],
    "component": {
      "type": "application",
      "bom-ref": "pkg:pypi/my-project@1.0.0",
      "name": "my-project",
      "version": "1.0.0"
    }
  },
  "components": [
    {
      "type": "library",
      "bom-ref": "pkg:pypi/requests@2.31.0",
      "name": "requests",
      "version": "2.31.0",
      "purl": "pkg:pypi/requests@2.31.0",
      "properties": [
        { "name": "cdx:dependency:type", "value": "direct" },
        { "name": "cdx:package-manager", "value": "pip" },
        { "name": "cdx:environment", "value": "default" }
      ]
    },
    {
      "type": "library",
      "bom-ref": "pkg:pypi/certifi@2024.2.2",
      "name": "certifi",
      "version": "2024.2.2",
      "purl": "pkg:pypi/certifi@2024.2.2",
      "properties": [
        { "name": "cdx:dependency:type", "value": "transitive" },
        { "name": "cdx:package-manager", "value": "pip" },
        { "name": "cdx:environment", "value": "default" }
      ]
    }
  ],
  "dependencies": [
    {
      "ref": "pkg:pypi/my-project@1.0.0",
      "dependsOn": ["pkg:pypi/requests@2.31.0"]
    }
  ]
}
```

### Package URL (purl) format

| Ecosystem | Format | Example |
|---|---|---|
| pip / uv / PyPI packages | `pkg:pypi/<name>@<version>` | `pkg:pypi/requests@2.31.0` |
| Pixi / conda (conda-forge) | `pkg:conda/conda-forge/<name>@<version>` | `pkg:conda/conda-forge/numpy@1.26.4` |
| Pixi / conda (defaults channel) | `pkg:conda/defaults/<name>@<version>` | `pkg:conda/defaults/python@3.13.0` |
| Pixi / conda (unknown channel) | `pkg:conda/<name>@<version>` | `pkg:conda/scipy@1.13.0` |

Package names are normalized to PEP 503 canonical form before being used in purls: lowercase, with `_` and `.` replaced by `-`.

### Component properties

Every component in the output carries three custom properties:

| Property name | Values | Meaning |
|---|---|---|
| `cdx:dependency:type` | `direct` / `transitive` | Whether the package appears in a manifest file (`direct`) or was pulled in as a transitive dependency (`transitive`) |
| `cdx:package-manager` | `pixi` / `uv` / `pip` / `conda` | The package manager that owns this component |
| `cdx:environment` | e.g., `default`, `py312`, `base` | The environment the component was found in — useful when diffing SBOMs across environments |

---

## Generation strategies

The skill selects one of three strategies based on what tooling is available:

### Strategy A — cyclonedx-py CLI

**When used:** `cyclonedx-py` is on PATH, `PKG_MANAGER` is `pip` or `uv`, and only a single environment is being generated.

```bash
cyclonedx-py environment \
  --output-format JSON \
  --outfile sbom.cyclonedx.json \
  --schema-version 1.4
```

After generation, a patch script adds the `cdx:dependency:type` and `cdx:environment` properties to each component.

Strategy A produces the most complete output because `cyclonedx-py` can resolve PEP 517 build metadata that the other strategies do not have access to.

### Strategy B — cyclonedx-python-lib

**When used:** `cyclonedx-python-lib` is importable in `GLOBAL_PYTHON` (the system Python or the pixi global env).

A Python script (embedded in `references/cyclonedx-generation.md`) uses `cyclonedx-python-lib` >= 8.0 to build and serialize the BOM programmatically. Falls back to Strategy C if the import fails.

**API compatibility note:** the `Component` constructor changed its `component_type=` keyword to `type=` at v8.0. The generation script uses the v8+ spelling. If a v7.x library is installed the strategy falls through to C automatically via the `ImportError` catch. See the version table in `references/cyclonedx-generation.md`.

This strategy works with any package manager and with multi-environment Pixi/conda.

### Strategy C — stdlib-only

**When used:** Always for JSON output, regardless of what other tooling is available. For XML, used when `cyclonedx-python-lib` is not importable.

Uses only Python's built-in `json`, `re`, `uuid`, and `datetime` modules to generate a valid CycloneDX 1.4 JSON (or minimal XML). No third-party dependency is required.

Strategy C is the primary JSON path — not a last resort. The library's JSON serializer produces different key ordering, non-deterministic `bom-ref` values, and missing metadata fields compared to the canonical format the skill defines. Using Strategy C for all JSON output guarantees identical structure across every project and package manager.

---

## Bootstrapping missing tools

If the manifest signals a package manager that is not installed, the skill installs it to a temporary directory, does the SBOM work, then deletes the directory. Nothing is written to system paths or the user's home directory.

### How it works

```bash
TEMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/sbom-bootstrap.XXXXXX")
# ... install tools here ...
# ... generate SBOMs ...
rm -rf "${TEMP_DIR}"   # always runs, even on failure
```

Each tool's `bin/` is prepended to `PATH` after installation so subsequent steps find it automatically. When `TEMP_DIR` is deleted in Step 7, those PATH entries become inert dead links — harmless since the shell session ends after the skill completes.

### What gets bootstrapped and from where

| Tool | Trigger | Source |
|---|---|---|
| **Pixi** | `pixi.toml` or `pixi.lock` found, `pixi` not on PATH | https://pixi.sh/install.sh (prefix.dev official installer) |
| **Miniforge (conda)** | `environment.yml` or `conda.yml` found, `conda` / `mamba` not on PATH | https://github.com/conda-forge/miniforge releases (Miniforge3 installer, `-b -p` batch mode) |
| **uv** | `uv.lock` or `[tool.uv]` found, `uv` not on PATH | https://astral.sh/uv/install.sh (Astral official installer) |
| **Python 3.13** | No `python3` on PATH, or version is < 3.8 | CPython 3.13 from python-build-standalone via `uv python install 3.13`; uv is bootstrapped first if also missing |
| **pip** | `requirements.txt` found, Python available, `pip` not found | `python -m ensurepip --upgrade --default-pip` (no download required) |
| **Poetry** | `poetry.lock` or `[tool.poetry]` found, `poetry` not on PATH | https://install.python-poetry.org (official installer, `POETRY_HOME` redirected to temp dir) |

Only missing tools are bootstrapped. The skill never re-installs a tool that is already on PATH.

### Bootstrapped tools in the report

The Step 6 report includes a "Bootstrapped (now removed)" section listing each temporarily installed tool and its source. This gives an audit trail of what ran during the SBOM generation session.

---

## Invoking from different tools

### Claude Code

```
/generate-python-sbom
/generate-python-sbom --path /path/to/project
/generate-python-sbom --path /path/to/project --format xml --env py312
```

The skill file at `.claude/skills/generate-python-sbom/SKILL.md` is loaded automatically when Claude Code recognizes the `/generate-python-sbom` command. Natural-language trigger phrases also work:

- "generate a CycloneDX SBOM for this project"
- "create an SBOM in XML format"
- "run a dependency audit"
- "export the package list for supply chain review"

### GitHub Copilot Agent

Open the workspace and prompt:

```
Follow the instructions in `.claude/skills/generate-python-sbom/SKILL.md` to generate a CycloneDX SBOM for the project at /path/to/project. Output format: JSON.
```

Reference files are loaded automatically because they are co-located with SKILL.md in the `.claude/skills/generate-python-sbom/` directory. If Copilot does not auto-include them, explicitly attach `SKILL.md`, `references/package-managers.md`, `references/cyclonedx-generation.md`, and `references/bootstrap-tools.md`.

### Devin

Paste the SKILL.md content into Devin's task description, or reference it:

```
Task: Follow the instructions in `.claude/skills/generate-python-sbom/SKILL.md` to generate a CycloneDX 1.4 SBOM.
Project: /path/to/project
Format: JSON
Include dev deps: no
```

Provide the three reference files as supplementary context if Devin cannot read the workspace directly.

### Any other LLM / agent

Provide as context (in order):

1. `SKILL.md` — the step-by-step instructions
2. `references/package-managers.md` — detection and query commands per manager
3. `references/cyclonedx-generation.md` — Python generation scripts
4. `references/bootstrap-tools.md` — per-tool bootstrap procedures

Then say: "Execute the generate-python-sbom skill for the project at `<path>`. Output format: `<json|xml>`."

Arguments can be natural language: "generate a CycloneDX SBOM in XML format for each environment, save to `/reports/`."

---

## Examples

### pip project with a .venv

```
/generate-python-sbom --path /projects/django-demo
```

Expected output:

```
Project root:     /projects/django-demo
Package manager:  pip (.venv detected at /projects/django-demo/.venv)
CycloneDX:        Strategy C (stdlib-only; cyclonedx-python-lib not available)

Generated SBOMs:
  /projects/django-demo/sbom.cyclonedx.json
  Components: 7 (5 direct, 2 transitive)

Direct deps found in manifest but missing from venv:  (none)

Validation (if cyclonedx-py is installed):
  cyclonedx validate --input-format JSON --input-version LATEST --input-file /projects/django-demo/sbom.cyclonedx.json
```

### Pixi project — all environments

```
/generate-python-sbom --path /projects/my-cli-tool
```

For a project with `default` and `py312` environments:

```
Project root:     /projects/my-cli-tool
Package manager:  pixi
CycloneDX:        Strategy C (stdlib-only)

Generated SBOMs:
  /projects/my-cli-tool/sbom.cyclonedx.default.json   67 components (2 direct, 65 transitive)
  /projects/my-cli-tool/sbom.cyclonedx.py312.json      64 components (2 direct, 62 transitive)
```

### Pixi project — one environment in XML

```
/generate-python-sbom --path /projects/my-cli-tool --env default --format xml
```

Output file: `/projects/my-cli-tool/sbom.cyclonedx.xml`

### conda project — bootstrap conda, then remove

If the project has `environment.yml` but `conda` is not installed:

```
/generate-python-sbom --path /projects/data-pipeline
```

```
conda not found on PATH. Bootstrapping Miniforge from conda-forge to /tmp/sbom-bootstrap.aX7q3b/conda ...
Miniforge installed. conda 23.11.0

... (SBOM generation) ...

Bootstrapped (now removed):
  conda (Miniforge)  — https://github.com/conda-forge/miniforge/releases/latest
```

### Specify a custom output directory

```
/generate-python-sbom --path /projects/my-api --output /reports/my-api-sbom --format json
```

Single env: `/reports/my-api-sbom.json`
Multi-env:  `/reports/my-api-sbom.default.json`, `/reports/my-api-sbom.py312.json`

---

## Validating the output

The `cyclonedx-py` binary installed by `cyclonedx-bom` generates SBOMs but does not include a `validate` subcommand. Schema validation requires the separate `cyclonedx-cli` tool (a standalone Go binary from CycloneDX):

```bash
# Install cyclonedx-cli (Go binary — not a Python package)
# macOS via Homebrew:
brew install cyclonedx/cyclonedx/cyclonedx-cli

# Or download from GitHub releases:
# https://github.com/CycloneDX/cyclonedx-cli/releases

# Validate JSON SBOM
cyclonedx validate --input-format json --input-version v1_4 --input-file sbom.cyclonedx.json

# Validate XML SBOM
cyclonedx validate --input-format xml --input-version v1_4 --input-file sbom.cyclonedx.xml
```

Exit 0 means the file is valid. Any output with a non-zero exit indicates a schema violation.

As a lightweight alternative, the skill performs structural validation inline using Python (checking required fields: `bomFormat`, `specVersion`, `serialNumber`, `components[].name/version/purl/type`) without requiring an external validator.

---

## File structure

```
.claude/skills/generate-python-sbom/
├── SKILL.md                          # Step-by-step instructions consumed by the LLM
├── README.md                         # This file — human-readable documentation
└── references/
    ├── cyclonedx-spec.md             # CycloneDX 1.4 schema reference (offline, no network needed)
    ├── package-managers.md           # Detection commands, manifest field locations, purl formats
    ├── cyclonedx-generation.md       # Python scripts for Strategies A, B, and C
    └── bootstrap-tools.md            # Per-tool temporary installation procedures
```

**SKILL.md** is the executable artifact. The LLM follows it step by step. It references the files under `references/` by name — those files contain lengthy command templates and scripts that would bloat SKILL.md if inlined.

**`references/cyclonedx-spec.md`** is the offline spec reference. It covers every field the skill reads or writes: required vs optional fields at the BOM/metadata/component level, component `type` enum values, purl format per ecosystem, the `properties` extension point, the `dependencies` graph, `bom-ref` rules, and common validation errors. No network access needed — the LLM can answer schema questions from this file alone.

**README.md** (this file) is for humans and for LLMs that need context before starting. It is not followed as instructions.

---

## Troubleshooting

### "No manifest found at PROJECT_ROOT"

The skill could not find any of: `pixi.toml`, `pyproject.toml`, `requirements.txt`, `environment.yml`, `setup.py`, `setup.cfg`, `uv.lock`, `pixi.lock` at the specified path.

- Confirm the path is correct: `ls /path/to/project`
- If auto-detection was used (no `--path`), re-run with `--path <correct-dir>`

### `conda env create` fails with "PackagesNotFoundError"

Some packages listed in `environment.yml` under `dependencies:` are PyPI-only and do not exist on conda-forge under the same name. Common examples: `build` (conda-forge name: `python-build`), `setuptools-scm`, and various dev tools.

When `--include-dev` is not set the skill automatically strips dev packages (those matching `pytest*`, `mypy`, `ruff`, `pre-commit`, `build`, `black`, `flake8`, `isort`, `coverage*`, `sphinx*`, `pandas-stubs`, `types-*`) from a temporary copy of the YAML before running `conda env create`. This avoids the error for the common case.

If a **runtime** package is causing the error, the environment YAML needs to be fixed — either move the package to the `pip:` sub-section or use its correct conda-forge name.

### `conda env create` fails or installs into the wrong directory when using `-e .`

A relative pip path (`- -e .`) in the `pip:` section of `environment.yml` resolves relative to the working directory at the time `conda env create` runs, not relative to the YAML file. Because the skill runs `conda env create` with `--prefix` pointing inside `TEMP_DIR`, the working directory may not be `PROJECT_ROOT`.

The skill rewrites any `- -e .` or `- -e <relative-path>` entries to absolute paths before creating the temporary environment. If generation fails with a pip install error mentioning the wrong path, confirm that `PROJECT_ROOT` was correctly detected.

### "Environment not installed" / package query fails

The environment exists in the manifest but has not been installed yet:

| Manager | Fix |
|---|---|
| Pixi | `cd <project-root> && pixi install` |
| uv | `uv sync` |
| conda | `conda env create -f environment.yml` |
| pip | `pip install -r requirements.txt` (inside a venv) |

### "pixi bootstrap failed" / "Miniforge bootstrap failed"

The temporary install failed — usually a network issue or missing `curl`/`wget`.

- Confirm internet access
- Confirm `curl` or `wget` is available: `which curl; which wget`
- Install the package manager manually and re-run the skill

### Direct dependencies missing from the installed environment

The skill reports a warning for any package listed in a manifest that is not present in the queried environment. Common causes:

- An optional extra (`requests[security]`) whose extra packages are not installed
- A platform-specific dependency that doesn't apply to the current OS
- A package installed with a different name than the manifest lists (rare; PEP 503 normalization handles most cases)

These are warnings only — the SBOM is still generated with all packages that are installed.

### CycloneDX tooling install failed and no stdlib fallback

If `pixi global install cyclonedx-bom` and `pip install --user cyclonedx-python-lib` both fail, the skill automatically falls back to the stdlib-only Strategy C. This fallback requires only `python3` and no network access.

If Strategy C also fails (no `python3` on PATH), bootstrap Python 3.13 by running with the intent to bootstrap Python:

```
/generate-python-sbom --path /path/to/project
```

The skill will detect that `python3` is missing and automatically bootstrap CPython 3.13 via uv.

### Generated SBOM is empty or missing

- Check Step 5a output — if the package query returned zero packages, the environment may be empty or the query command failed silently
- For pip: confirm the venv is at `.venv/` or `venv/` inside the project root, or that `pip` on PATH points to the right environment
- For Pixi: confirm the environment is installed (`pixi install --environment <name>`)
