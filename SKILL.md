---
name: generate-python-sbom
description: >-
  Generate a CycloneDX SBOM (Software Bill of Materials) for a Python project. Detects Pixi, uv, pip, conda, and Poetry automatically. Bootstraps any missing package manager or Python runtime to a temporary directory, does the work, then removes it. For Pixi and conda, generates one SBOM per environment unless a specific environment is requested. Extracts direct dependencies from manifest files (pixi.toml, pyproject.toml, requirements.txt, environment.yml, etc.) and transitive dependencies from the installed environment. Outputs CycloneDX 1.4 JSON or XML. Trigger on: "generate SBOM", "generate Python SBOM", "create SBOM", "software bill of materials", "cyclonedx", "export dependencies as SBOM", "dependency audit", "supply chain", "list all packages with versions".
argument-hint: "[--path <project-dir>] [--format json|xml] [--output <path>] [--env <name>] [--include-dev]"
compatibility: Pixi · uv · pip · conda · Python 3.8+ · CycloneDX 1.4
disable-model-invocation: false
license: MIT
metadata:
  author: Kevin Mills
  tags: sbom, cyclonedx, security, supply-chain, pixi, uv, pip, conda, python, compliance, generate-python-sbom
user-invokable: true
---

# generate-python-sbom

Generate a CycloneDX Software Bill of Materials (SBOM) for a Python project. Package-manager agnostic — supports Pixi, uv, pip, and conda. For Pixi and conda, generates one SBOM per environment by default; use `--env` to limit to a single environment. CycloneDX tooling is installed globally so no project environment is modified.

This skill is written as self-contained, numbered steps that can be executed by any LLM-powered tool (Claude Code, GitHub Copilot Agent, Devin, etc.) — see **Compatibility** at the bottom for tool-specific invocation notes.

Reference files in `references/`:
- `cyclonedx-spec.md` — CycloneDX 1.4 schema: required fields, component types, purl format, dependency graph, validation
- `package-managers.md` — detection commands, manifest field locations, purl rules per manager
- `cyclonedx-generation.md` — Python generation scripts (Strategies A, B, C)
- `bootstrap-tools.md` — always-bootstrap procedures; references installation-sources.md and config files
- `installation-sources.md` — **swap this file** to change download URLs for private/corporate environments
- `pip.conf` — pip configuration written to TEMP_DIR before every pip invocation
- `condarc` — conda configuration written to TEMP_DIR before every conda invocation
- `uv-config.toml` — uv configuration written to TEMP_DIR before every uv invocation
- `pixi-config.toml` — pixi configuration written to TEMP_DIR before every pixi invocation

---

## Arguments

`$ARGUMENTS` is parsed as: `[--path <project-dir>] [--format json|xml] [--output <path>] [--env <name>] [--include-dev]`

| Argument | Default | Description |
|---|---|---|
| `--path` | current working directory | Absolute or relative path to the Python project root |
| `--format` | `json` | Output format: `json` or `xml` |
| `--output` | `<project-root>/sbom.cyclonedx[.<env>].<format>` | Output file path; env suffix is added automatically when multiple environments are generated |
| `--env` | _(all environments)_ | For Pixi/conda: limit generation to this one named environment. For pip/uv: unused. |
| `--include-dev` | false | Include dev/test/optional dependency groups in the direct-dependency list |

Resolve variables before starting:

- `PROJECT_ROOT` = absolute path of `--path` value if present, else `pwd`. Resolve immediately — use this in every file operation below, never bare relative paths.
- `FORMAT` = `json` unless `--format xml` is in `$ARGUMENTS`
- `OUTPUT_BASE` = value of `--output` if present, else `${PROJECT_ROOT}/sbom.cyclonedx`
- `ENV_FILTER` = value of `--env` if present, else empty string (meaning: all environments)
- `INCLUDE_DEV` = `true` if `--include-dev` in `$ARGUMENTS`, else `false`

If `--path` is supplied, confirm the directory exists before proceeding:

```bash
test -d "${PROJECT_ROOT}" || { echo "ERROR: project root not found: ${PROJECT_ROOT}"; exit 1; }
```

---

## Step 1 — Detect package manager and global tooling

Run these checks in parallel:

```bash
# Manifest files
ls "${PROJECT_ROOT}/pixi.toml" "${PROJECT_ROOT}/pyproject.toml" "${PROJECT_ROOT}/requirements.txt" \
   "${PROJECT_ROOT}/environment.yml" "${PROJECT_ROOT}/conda.yml" "${PROJECT_ROOT}/setup.py" \
   "${PROJECT_ROOT}/setup.cfg" "${PROJECT_ROOT}/uv.lock" "${PROJECT_ROOT}/pixi.lock" 2>/dev/null

# Available tools
which pixi uv conda pip python3 2>/dev/null
```

Assign `PKG_MANAGER` using this priority order (first match wins):

1. **pixi** — `${PROJECT_ROOT}/pixi.toml` exists
2. **uv** — `${PROJECT_ROOT}/uv.lock` exists, OR `${PROJECT_ROOT}/pyproject.toml` contains `[tool.uv]`
3. **conda** — `${PROJECT_ROOT}/environment.yml` or `${PROJECT_ROOT}/conda.yml` exists, OR `conda` is on PATH and no `pixi.toml` is present
4. **pip** — `${PROJECT_ROOT}/requirements.txt` (or `requirements-*.txt`) exists, OR `setup.py` / `setup.cfg` / `pyproject.toml` without uv config

If no manifest is found at `PROJECT_ROOT`, stop and tell the user. If `--path` was not supplied, suggest re-running with `--path <correct-dir>`. The presence or absence of tools on the system PATH does not affect execution — all tools are always bootstrapped fresh in Step 1b.

---

## Step 1b — Bootstrap tools to TEMP_DIR

Tools are **always** bootstrapped to a fresh `TEMP_DIR` for every run, regardless of what is installed on the machine. This guarantees isolation: no project files are touched, no user home directories are modified, and every run resolves packages from a clean state.

**Idempotency:** Running this skill more than once against the same project is safe. SBOM output files are overwritten in-place on each run. Each run uses a fresh `TEMP_DIR` that is always cleaned up on exit, regardless of success or failure. If a previous run was interrupted and left a stale temp directory, the janitor step below removes it.

Download URLs, version pins, and package index configuration are read from the reference files in `references/`. To adapt for a private or corporate environment, replace those files — see [`references/installation-sources.md`](references/installation-sources.md) for the complete guide.

```bash
# Janitor: remove stale sbom-bootstrap temp dirs older than 1 hour from any
# previous interrupted run. Guard with 2>/dev/null so a missing find flag
# on older macOS doesn't abort. Ignore errors — this is best-effort.
find "${TMPDIR:-/tmp}" -maxdepth 1 -name "sbom-bootstrap.*" -type d \
  -mmin +60 2>/dev/null | xargs rm -rf 2>/dev/null || true

# Cleanup trap — fires on EXIT (success or error), INT (Ctrl-C), TERM, and QUIT.
# This is the primary guarantee that TEMP_DIR is never left behind.
_sbom_cleanup() {
  for _prefix in "${CONDA_TEMP_PREFIXES[@]:-}"; do
    [ -n "${_prefix}" ] && [ -d "${_prefix}" ] && rm -rf "${_prefix}"
  done
  [ -n "${TEMP_DIR}" ] && [ -d "${TEMP_DIR}" ] && rm -rf "${TEMP_DIR}"
}
trap '_sbom_cleanup' EXIT INT TERM QUIT

TEMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/sbom-bootstrap.XXXXXX")
BOOTSTRAPPED_TOOLS=()
CONDA_TEMP_PREFIXES=()   # tracks conda env prefixes for Step 7a cleanup

Bootstrap the tool for `PKG_MANAGER` following the exact procedures in [`references/bootstrap-tools.md`](references/bootstrap-tools.md):

| PKG_MANAGER | Tool bootstrapped | Where |
|---|---|---|
| `pip` | uv → Python 3.13 | `${TEMP_DIR}/uv`, `${TEMP_DIR}/uv/python` |
| `uv` | uv → Python 3.13 | `${TEMP_DIR}/uv`, `${TEMP_DIR}/uv/python` |
| `conda` | Miniforge | `${TEMP_DIR}/conda` |
| `pixi` | pixi + uv → Python 3.13 | `${TEMP_DIR}/pixi`, `${TEMP_DIR}/uv` |

After bootstrapping the project tool, `SBOM_PYTHON` is set to the Python that will be used for SBOM generation (the Strategy B/C scripts). After bootstrapping completes, always:

```bash
# Install cyclonedx-bom into SBOM_PYTHON — see bootstrap-tools.md §CycloneDX
"${SBOM_PYTHON}" -m pip install --quiet cyclonedx-bom
```

Verify the bootstrap succeeded before continuing:

```bash
"${SBOM_PYTHON}" -c "import cyclonedx; print('cyclonedx-bom', cyclonedx.__version__)" \
  || { echo "ERROR: cyclonedx-bom install failed"; exit 1; }
```

If any bootstrap step fails, stop and surface the exact error. Do not proceed with partial tooling.

After bootstrapping, prepend the tool's `bin/` directory to `PATH` so all subsequent steps use the TEMP_DIR binaries rather than any system binaries:

```bash
export PATH="${TEMP_DIR}/uv/bin:${TEMP_DIR}/pixi/bin:${TEMP_DIR}/conda/bin:${PATH}"
```

---

## Step 2 — Extract direct dependencies from manifests

Read every applicable manifest file under `PROJECT_ROOT` to build `DIRECT_DEPS` — a list of canonical package names. See `references/package-managers.md` for exact field locations per file type.

**`${PROJECT_ROOT}/pixi.toml`** — use `tomllib` (Python 3.11+) or `tomli`; never regex-split on `[section]` headers because comments can contain bracket strings that break regex-based parsing:

```python
try:
    import tomllib
except ImportError:
    import tomli as tomllib   # pip install tomli on Python < 3.11

with open(f"{PROJECT_ROOT}/pixi.toml", "rb") as f:
    toml = tomllib.load(f)

# Base runtime deps (always included)
direct = list(toml.get("dependencies", {}).keys()) + list(toml.get("pypi-dependencies", {}).keys())

# Dev/feature deps (only when INCLUDE_DEV=true)
if INCLUDE_DEV:
    for feat in toml.get("feature", {}).values():
        direct += list(feat.get("dependencies", {}).keys())

# Exclude 'python' itself and normalize
direct = [re.sub(r"[-_.]+", "-", d).lower() for d in direct if d.lower() != "python"]
```

**`${PROJECT_ROOT}/pyproject.toml`** — similarly use `tomllib`:

```python
with open(f"{PROJECT_ROOT}/pyproject.toml", "rb") as f:
    toml = tomllib.load(f)

project = toml.get("project", {})
direct = [re.split(r"[>=<!;\[\s]", dep)[0].strip() for dep in project.get("dependencies", [])]

if INCLUDE_DEV:
    for extras in project.get("optional-dependencies", {}).values():
        direct += [re.split(r"[>=<!;\[\s]", dep)[0].strip() for dep in extras]
    tool_uv = toml.get("tool", {}).get("uv", {})
    direct += [re.split(r"[>=<!;\[\s]", dep)[0].strip() for dep in tool_uv.get("dev-dependencies", [])]
```

**`${PROJECT_ROOT}/requirements.txt` / `requirements-*.txt` / `constraints.txt`**
- Each non-comment, non-empty, non-`-r`-include line is a direct dep

**conda — all `environment.yml` / `conda.yml` files under `PROJECT_ROOT`**

For conda projects, Step 2 discovers *all* conda environment YAML files in the project tree (root and subdirectories) and parses direct deps from each one. This builds `CONDA_ENV_FILES` — the authoritative source for Step 4.

```python
import os, re

CONDA_ENV_FILES = []   # list of {yaml_path, name, direct_names}

CONDA_YAML_PATTERN = re.compile(
    r"^(environment|conda)([.\-].+)?\.ya?ml$", re.IGNORECASE
)

for root, dirs, files in os.walk(PROJECT_ROOT):
    # skip hidden dirs and common non-project dirs
    dirs[:] = [d for d in dirs if not d.startswith(".") and d not in ("__pycache__", "node_modules")]
    for fname in sorted(files):
        if not CONDA_YAML_PATTERN.match(fname):
            continue
        yaml_path = os.path.join(root, fname)

        # Section-aware YAML parse (no PyYAML dependency)
        with open(yaml_path) as f:
            lines = f.readlines()

        section = None
        in_pip = False
        env_name = ""
        conda_deps = []
        pip_deps = []

        for line in lines:
            stripped = line.strip()
            if not stripped or stripped.startswith("#"):
                continue
            # top-level key (no leading indent)
            if re.match(r"^[a-z_]", line):
                m = re.match(r"^([a-z_]+):\s*(.*)", line)
                if m:
                    section = m.group(1)
                    if section == "name":
                        env_name = m.group(2).strip()
                in_pip = False
                continue
            if section == "dependencies":
                if stripped == "- pip:":
                    in_pip = True
                    continue
                if in_pip:
                    if stripped.startswith("- "):
                        val = stripped[2:].strip()
                        if not val.startswith("-e") and not val.startswith("--"):
                            pip_deps.append(re.split(r"[>=<!;\[\s]", val)[0].strip())
                    else:
                        in_pip = False
                if not in_pip and stripped.startswith("- "):
                    dep_val = stripped[2:].strip()
                    if dep_val.startswith("#"):
                        continue
                    pkg = re.split(r"[>=<!;\[\s]", dep_val)[0].strip()
                    if pkg.lower() not in ("python", "pip", ""):
                        conda_deps.append(pkg)

        # Normalize: exclude dev-only packages unless INCLUDE_DEV
        DEV_PKG_PREFIXES = ("pytest", "mypy", "ruff", "pre-commit", "build",
                             "black", "flake8", "isort", "coverage", "sphinx",
                             "pandas-stubs", "types-", "pytest-")
        runtime_deps = [
            d for d in conda_deps + pip_deps
            if not (not INCLUDE_DEV and any(d.lower().startswith(p) for p in DEV_PKG_PREFIXES))
        ]
        direct_names = [re.sub(r"[-_.]+", "-", d).lower() for d in runtime_deps if d]

        CONDA_ENV_FILES.append({
            "yaml_path": yaml_path,
            "name": env_name or os.path.splitext(fname)[0],
            "direct_names": direct_names,
        })

# INCLUDE_DEV is false by default — see argument parsing at the top
```

**`${PROJECT_ROOT}/setup.py` / `setup.cfg`**
- Read `install_requires`; only if `pyproject.toml` was not already parsed

Normalize all names to lowercase with hyphens (PEP 503): `re.sub(r"[-_.]+", "-", name).lower()`.

---

## Step 3 — Verify SBOM generation tooling

CycloneDX tooling was installed in Step 1b into `SBOM_PYTHON`. This step is a fast verification gate — it does not install anything.

```bash
"${SBOM_PYTHON}" -c "import cyclonedx; print('cyclonedx-bom', cyclonedx.__version__)" \
  || { echo "ERROR: cyclonedx-bom not importable from ${SBOM_PYTHON} — check Step 1b output"; exit 1; }
```

`SBOM_PYTHON` is the Python used for all Strategy B/C generation scripts in `references/cyclonedx-generation.md`. It lives entirely inside `${TEMP_DIR}` and is never a project-environment or system Python.

If verification fails, re-run Step 1b rather than attempting an install here. This keeps the single-responsibility boundary clean: Step 1b owns all installs, Step 3 only asserts readiness.

---

## Step 4 — Enumerate target environments

Build `ENV_LIST` — the ordered list of `{name, query_spec}` pairs to generate SBOMs for.

### Pixi

Read the `[environments]` table from `${PROJECT_ROOT}/pixi.toml`. Use `tomllib` — regex-based section splitting is unreliable when comments contain bracket strings.

```python
try:
    import tomllib
except ImportError:
    import tomli as tomllib

with open(f"{PROJECT_ROOT}/pixi.toml", "rb") as f:
    toml = tomllib.load(f)

env_names = list(toml.get("environments", {}).keys()) or ["default"]
```

If `pixi.toml` has no `[environments]` table, `env_names` defaults to `["default"]`.

If `ENV_FILTER` is non-empty, verify it exists in the list then set `ENV_LIST = [{name: ENV_FILTER}]`. If it does not exist, stop and report available environments.

Each entry's `query_spec` = `--environment <name>` for the `pixi list` command.

### conda

`CONDA_ENV_FILES` was built in Step 2. Each entry is a `{yaml_path, name, direct_names}` dict. `ENV_LIST` for conda is derived directly from those files — **not** from `conda info --json`. This keeps the SBOM scoped to the project's own declared environments, regardless of what other conda environments happen to be installed on the machine.

```python
# ENV_LIST for conda — one entry per discovered YAML file
ENV_LIST = [
    {
        "name": entry["name"],
        "yaml_path": entry["yaml_path"],
        "direct_names": entry["direct_names"],
    }
    for entry in CONDA_ENV_FILES
]
```

If `CONDA_ENV_FILES` is empty, stop and tell the user: no conda environment YAML files were found under `PROJECT_ROOT`. Confirm that `PKG_MANAGER` detection is correct.

If `ENV_FILTER` is non-empty, filter `ENV_LIST` to only the entry whose `name` matches. If no match, stop and list the discovered environment names.

Each entry's `query_spec` for Step 5a is `--name ${TEMP_ENV_NAME}` where `TEMP_ENV_NAME` is created in Step 5a-pre.

### pip / uv

`ENV_LIST` is always a single entry: `[{name: "default", query_spec: ""}]`. The `--env` argument is not applicable and is ignored with a warning if supplied.

---

## Step 5 — Generate one SBOM per environment

For each `{name, query_spec}` in `ENV_LIST`, run the following sub-steps. If `ENV_LIST` has more than one entry, suffix the output filename with the environment name.

### Resolve output filename for this environment

```
SINGLE_ENV = (length of ENV_LIST == 1)
if SINGLE_ENV:
    OUTPUT = "${OUTPUT_BASE}.${FORMAT}"
else:
    OUTPUT = "${OUTPUT_BASE}.${name}.${FORMAT}"
    # e.g. sbom.cyclonedx.default.json, sbom.cyclonedx.py312.json
```

### 5a-pre — Create project environment in TEMP_DIR (pip / uv / conda)

**Skip this sub-step for pixi** — pixi manages its own project-local environments; Step 5a queries them directly.

#### pip / uv

Create a fresh virtualenv inside `TEMP_DIR` and install the project's dependencies into it. This is the source of the full resolved package list (direct + transitive).

```bash
VENV_DIR="${TEMP_DIR}/venv"

# Create the venv using SBOM_PYTHON (the bootstrapped Python from Step 1b)
"${SBOM_PYTHON}" -m venv "${VENV_DIR}"
VENV_PIP="${VENV_DIR}/bin/pip"

# Apply pip config so the install uses the correct index
export PIP_CONFIG_FILE="${TEMP_DIR}/pip/pip.conf"

echo "Installing project dependencies into ${VENV_DIR} ..."
# Detect which manifest to install from
if [ -f "${PROJECT_ROOT}/uv.lock" ] || [ -f "${PROJECT_ROOT}/pyproject.toml" ]; then
  # uv project: use uv to install with locked resolution
  uv pip install --python "${VENV_DIR}" \
    -r "${PROJECT_ROOT}/pyproject.toml" --all-extras --quiet 2>/dev/null \
  || "${VENV_PIP}" install "${PROJECT_ROOT}" --quiet
elif [ -f "${PROJECT_ROOT}/requirements.txt" ]; then
  "${VENV_PIP}" install -r "${PROJECT_ROOT}/requirements.txt" --quiet
fi
```

If the install fails, report the error for this environment and skip it.

#### conda

Create the environment with `--prefix` inside `TEMP_DIR`. Using `--prefix` (not `--name`) keeps the environment entirely within `TEMP_DIR` and adds no entry to the user's conda environment registry.

```bash
YAML_PATH="${entry[yaml_path]}"
ENV_SLUG=$(echo "${entry[name]}" | tr '[:upper:]' '[:lower:]' | tr -cs 'a-z0-9' '-')
CONDA_ENV_PREFIX="${TEMP_DIR}/conda-envs/${ENV_SLUG}"
mkdir -p "${TEMP_DIR}/conda-envs"

echo "Creating conda environment from ${YAML_PATH} ..."
echo "(This may take several minutes while conda resolves and downloads packages)"

conda env create \
  --file "${YAML_PATH}" \
  --prefix "${CONDA_ENV_PREFIX}" \
  --quiet

if [ $? -ne 0 ]; then
  echo "ERROR: conda env create failed for ${YAML_PATH} — skipping"
  continue
fi

# Track prefix for Step 5-post and Step 7a safety cleanup
CONDA_TEMP_PREFIXES+=("${CONDA_ENV_PREFIX}")
```

### 5a — Query installed packages

Use the appropriate command for `PKG_MANAGER`. All environments are inside `TEMP_DIR` (except pixi, which queries project-local envs without modifying them).

| Manager | Query command |
|---|---|
| pip | `"${TEMP_DIR}/venv/bin/pip" list --format json` |
| uv | `uv pip list --format json --python "${TEMP_DIR}/venv"` |
| conda | `conda list --prefix "${CONDA_ENV_PREFIX}" --json` |
| pixi | `pixi list --json --environment "${ENV_NAME}" --manifest-path "${PROJECT_ROOT}/pixi.toml"` |

If the query command fails, report the error for this environment, skip it, and continue with the next entry in `ENV_LIST`. After the loop, if all environments failed, surface the errors.

Store the result as `INSTALLED_PACKAGES` — the raw JSON array from the command.

### 5b — Generate the SBOM

Choose the generation strategy in priority order:

| Strategy | Condition | Notes |
|---|---|---|
| **A — cyclonedx-py CLI** | `cyclonedx-py` is on PATH AND `PKG_MANAGER` is `pip` or `uv` AND the venv is activated AND only one environment AND `FORMAT` is `xml` | CLI runs against the active env; cannot target arbitrary envs. For JSON, skip to B/C even when the CLI is available. |
| **B — cyclonedx-python-lib (XML only)** | `FORMAT` is `xml` AND `GLOBAL_PYTHON` can import `cyclonedx` AND `USE_STDLIB_ONLY` is not set | Library serializer is only used for XML — its namespace and entity handling is more correct than hand-rolled XML. For JSON, always use C. |
| **C — stdlib-only script** | All JSON output; XML when library is unavailable | Produces the canonical output structure: deterministic purl-based `bom-ref`, logical key order, `metadata.tools`, `cdx:environment` on every component. All JSON SBOMs use this path regardless of strategy, ensuring identical format across all projects and package managers. |

**Strategy A:**
```bash
cyclonedx-py environment \
  --output-format "${FORMAT^^}" \
  --outfile "${OUTPUT}" \
  --schema-version 1.4
```
Then patch direct-dep annotations using the patch script in `references/cyclonedx-generation.md`.

**Strategy B / C:**
Use `GLOBAL_PYTHON` (never a project-environment Python) to run the generation script from `references/cyclonedx-generation.md`, passing:
- `INSTALLED_PACKAGES_JSON` — the raw JSON string from Step 5a
- `DIRECT_DEP_NAMES_JSON` — JSON array of canonical names from Step 2
- `FORMAT`, `OUTPUT`, `PKG_MANAGER`, `ENV_NAME` = current `{name}`
- `PROJECT_NAME` — from the manifest (`name` field in pixi.toml / pyproject.toml)
- `PROJECT_VERSION` — from the manifest, else `0.0.0`
- `USE_STDLIB_ONLY` — `true` or `false`

### 5c — Verify this environment's SBOM

```bash
test -s "${OUTPUT}" && echo "OK: ${OUTPUT}" || echo "ERROR: ${OUTPUT} is empty or missing"
```

For JSON, print a quick summary:
```bash
"${GLOBAL_PYTHON}" - <<'PY'
import json
with open("${OUTPUT}") as f:
    bom = json.load(f)
components = bom.get("components", [])
direct = [c for c in components if any(p.get("value") == "direct" for p in c.get("properties", []) if p.get("name") == "cdx:dependency:type")]
print(f"  {len(components)} components ({len(direct)} direct, {len(components)-len(direct)} transitive)")
PY
```

### 5-post — Remove temporary conda environment prefix (conda only)

**Skip this sub-step for non-conda projects.**

Remove the conda environment prefix created in 5a-pre immediately after the SBOM for this entry is written. Run this even if 5b or 5c failed — the temp environment should never be left behind.

Because environments are created with `--prefix` (not `--name`), removal is a simple directory delete — no `conda env remove` needed, and the user's conda registry is not touched.

```bash
if [ -n "${CONDA_ENV_PREFIX}" ] && [ -d "${CONDA_ENV_PREFIX}" ]; then
  echo "Removing temporary conda env: ${CONDA_ENV_PREFIX}"
  rm -rf "${CONDA_ENV_PREFIX}"
  # Remove from tracking list so Step 7a knows it's already gone
  CONDA_TEMP_PREFIXES=("${CONDA_TEMP_PREFIXES[@]/$CONDA_ENV_PREFIX}")
  CONDA_ENV_PREFIX=""
fi
```

---

## Step 6 — Report

After the loop completes, report a summary:

1. **Project root** used (`PROJECT_ROOT`)
2. **Package manager** detected
3. **CycloneDX tooling** used (Strategy A/B/C and whether a global install was performed)
4. For **each generated SBOM** (one row per environment):
   - Environment name
   - Absolute output file path
   - Component count (direct / transitive)
   - Source YAML file (conda only — the `environment.yml` the temp env was created from)
   - Any manifest deps not found in the installed environment (warnings)
5. If any environments were **skipped** due to errors, list them with the error message
6. **Bootstrapped tools** (if `BOOTSTRAPPED_TOOLS` is non-empty): list each tool and its source URL, noting it was installed temporarily and will be removed in Step 7
7. Validation command (requires `cyclonedx-cli` Go binary — separate from `cyclonedx-py`):
   ```
   cyclonedx validate --input-format ${FORMAT} --input-version v1_4 --input-file <output-file>
   ```
   Install: `brew install cyclonedx/cyclonedx/cyclonedx-cli` or download from https://github.com/CycloneDX/cyclonedx-cli/releases. If not available, the structural field checks in Step 5c serve as the validation pass.

---

## Step 7 — Cleanup

The `_sbom_cleanup` trap registered in Step 1b fires automatically on EXIT, INT, TERM, and QUIT — so cleanup runs regardless of success, failure, or user interruption. Steps 7a and 7b are the explicit equivalents of that trap; they serve as documentation of what the trap does and as a fallback for execution contexts (e.g., an LLM running steps interactively) where bash traps are not in effect.

### 7a — Remove surviving conda env prefixes

Each conda env prefix is normally deleted in Step 5-post immediately after its SBOM is written. This handles any that survived due to an early exit or error.

```bash
for prefix in "${CONDA_TEMP_PREFIXES[@]:-}"; do
  [ -n "${prefix}" ] && [ -d "${prefix}" ] && rm -rf "${prefix}" \
    && echo "Removed conda env prefix: ${prefix}"
done
CONDA_TEMP_PREFIXES=()
```

### 7b — Remove TEMP_DIR

```bash
if [ -n "${TEMP_DIR}" ] && [ -d "${TEMP_DIR}" ]; then
  echo "Removing TEMP_DIR: ${TEMP_DIR}"
  rm -rf "${TEMP_DIR}"
  unset TEMP_DIR
fi
```

This removes all bootstrapped binaries, Python installations, venvs, conda environments, caches, and installer artifacts in a single operation. Nothing outside `TEMP_DIR` was modified during the run (SBOM output files in `PROJECT_ROOT` are the sole intended artifacts).

If `BOOTSTRAPPED_TOOLS` is non-empty, include the list in the Step 6 report under a "Bootstrapped (removed)" heading so the user knows what was temporarily installed.

After Step 7 completes, the only files added to the filesystem are the SBOM output file(s) at the paths reported in Step 6.

---

## Compatibility

This skill's step-by-step instructions are tool-agnostic markdown. Use them with:

| Tool | How to invoke |
|---|---|
| **Claude Code** | `/generate-python-sbom [args]` or trigger phrase (e.g., "generate a cyclonedx SBOM") |
| **GitHub Copilot Agent** | Open the SKILL.md in the workspace and prompt: "Follow the instructions in `.claude/skills/generate-python-sbom/SKILL.md` to generate a CycloneDX SBOM" |
| **Devin** | Paste the SKILL.md content into Devin's task description, or reference it with "Follow the instructions in `.claude/skills/generate-python-sbom/SKILL.md`" |
| **Generic LLM** | Provide the SKILL.md + both reference files as context, then say "execute the generate-python-sbom skill" |

Arguments can be passed as natural language for non-Claude-Code tools: "generate a CycloneDX SBOM for the project at /path/to/project, one per environment, in XML format."
