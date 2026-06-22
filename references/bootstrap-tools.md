# Bootstrap Tools Reference

Tools are **always** bootstrapped to a fresh `TEMP_DIR` regardless of what is already installed on the machine. This guarantees a reproducible, isolated run that never reads from or writes to project files, user home directories, or system paths.

All download URLs, version pins, and package index configurations are sourced from [`installation-sources.md`](installation-sources.md) and the per-manager config files in this directory. Replace those files to adapt to private or corporate environments.

---

## Shared setup (always runs first)

```bash
# Create an isolated working directory for this entire SBOM generation session
TEMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/sbom-bootstrap.XXXXXX")

# Platform detection
OS=$(uname -s | tr '[:upper:]' '[:lower:]')    # darwin | linux
ARCH=$(uname -m)                                # x86_64 | arm64 | aarch64
# Note: macOS uses arm64; Linux uses aarch64. Do NOT normalise between them —
# Miniforge asset filenames follow the platform convention, not a unified scheme.

# Prefer curl; fall back to wget
if command -v curl >/dev/null 2>&1; then
  FETCH="curl -fsSL"
elif command -v wget >/dev/null 2>&1; then
  FETCH="wget -qO-"
else
  echo "ERROR: neither curl nor wget available. Cannot bootstrap tools."
  exit 1
fi

# Resolve the per-manager config source directory.
# Set SBOM_SKILL_SOURCES_DIR to override with private/corporate configs.
SKILL_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SOURCES_DIR="${SBOM_SKILL_SOURCES_DIR:-${SKILL_DIR}}"
```

---

## uv (always bootstrapped when PKG_MANAGER is uv OR when Python is needed)

uv is the meta-bootstrapper: it installs Python and manages virtualenvs. It is bootstrapped whenever PKG_MANAGER is `uv` **and** whenever Python is needed for SBOM generation (which is always).

```bash
UV_DIR="${TEMP_DIR}/uv"
mkdir -p "${UV_DIR}/bin"

echo "Bootstrapping uv to ${UV_DIR} ..."
${FETCH} https://astral.sh/uv/install.sh | UV_INSTALL_DIR="${UV_DIR}/bin" sh

# Point uv at the config file from references/ (or the private override)
UV_CONFIG="${SOURCES_DIR}/uv-config.toml"
export UV_CONFIG_FILE="${UV_CONFIG}"

# Keep all uv state (cache, Python installations) inside TEMP_DIR
export UV_CACHE_DIR="${UV_DIR}/cache"
export UV_PYTHON_INSTALL_DIR="${UV_DIR}/python"
export UV_DATA_DIR="${UV_DIR}/data"

export PATH="${UV_DIR}/bin:${PATH}"
uv --version || { echo "ERROR: uv bootstrap failed"; exit 1; }

# Install Python 3.13 for SBOM generation
uv python install 3.13
SBOM_PYTHON="$(uv python find 3.13)"
echo "SBOM Python: ${SBOM_PYTHON}"
```

**Cleanup:** removed automatically when `rm -rf "${TEMP_DIR}"` runs.

---

## Pixi (bootstrapped when PKG_MANAGER is pixi)

```bash
PIXI_INSTALL_DIR="${TEMP_DIR}/pixi"
mkdir -p "${PIXI_INSTALL_DIR}/bin"

echo "Bootstrapping pixi to ${PIXI_INSTALL_DIR} ..."
${FETCH} https://pixi.sh/install.sh | PIXI_HOME="${PIXI_INSTALL_DIR}" bash

# Apply config from references/pixi-config.toml (or private override)
PIXI_CONFIG_SRC="${SOURCES_DIR}/pixi-config.toml"
mkdir -p "${PIXI_INSTALL_DIR}"
cp "${PIXI_CONFIG_SRC}" "${PIXI_INSTALL_DIR}/config.toml"

export PIXI_HOME="${PIXI_INSTALL_DIR}"
export PATH="${PIXI_INSTALL_DIR}/bin:${PATH}"
pixi --version || { echo "ERROR: pixi bootstrap failed"; exit 1; }

# Python for SBOM generation — use uv (always bootstrapped alongside pixi)
# (bootstrap uv first if not already done in this session)
```

**Cleanup:** removed automatically when `rm -rf "${TEMP_DIR}"` runs.

---

## Miniforge / conda (bootstrapped when PKG_MANAGER is conda)

```bash
TEMP_CONDA_DIR="${TEMP_DIR}/conda"

# Build installer filename — macOS uses arm64, Linux uses aarch64
case "${OS}" in
  darwin)
    INSTALLER_OS="MacOSX"
    INSTALLER_ARCH="${ARCH}"       # arm64 on Apple Silicon, x86_64 on Intel
    ;;
  linux)
    INSTALLER_OS="Linux"
    # Normalise arm64 → aarch64 for Linux asset naming
    [ "${ARCH}" = "arm64" ] && INSTALLER_ARCH="aarch64" || INSTALLER_ARCH="${ARCH}"
    ;;
  *)
    echo "ERROR: unsupported OS for Miniforge: ${OS}"
    exit 1
    ;;
esac

INSTALLER_NAME="Miniforge3-${INSTALLER_OS}-${INSTALLER_ARCH}.sh"
INSTALLER_URL="https://github.com/conda-forge/miniforge/releases/latest/download/${INSTALLER_NAME}"
MINIFORGE_INSTALLER="${TEMP_DIR}/miniforge_install.sh"

echo "Bootstrapping Miniforge (${INSTALLER_NAME}) to ${TEMP_CONDA_DIR} ..."
${FETCH} "${INSTALLER_URL}" > "${MINIFORGE_INSTALLER}"
chmod +x "${MINIFORGE_INSTALLER}"

# Install in non-interactive batch mode — nothing written outside TEMP_DIR
bash "${MINIFORGE_INSTALLER}" -b -p "${TEMP_CONDA_DIR}"

# Apply conda config from references/condarc (or private override)
CONDARC_SRC="${SOURCES_DIR}/condarc"
# Expand ${TEMP_CONDA_DIR} placeholder in the template
sed "s|\${TEMP_CONDA_DIR}|${TEMP_CONDA_DIR}|g" "${CONDARC_SRC}" \
  > "${TEMP_CONDA_DIR}/.condarc"
export CONDARC="${TEMP_CONDA_DIR}/.condarc"

# Source conda for this shell session only
source "${TEMP_CONDA_DIR}/etc/profile.d/conda.sh"
export PATH="${TEMP_CONDA_DIR}/bin:${PATH}"
conda --version || { echo "ERROR: Miniforge bootstrap failed"; exit 1; }

# Python for SBOM generation — use Miniforge's base Python
SBOM_PYTHON="${TEMP_CONDA_DIR}/bin/python3"
```

**Cleanup:** removed automatically when `rm -rf "${TEMP_DIR}"` runs. Unlike named conda environments, using `--prefix` in TEMP_DIR means no entry is added to the user's conda environment registry.

---

## Python (bootstrapped when PKG_MANAGER is pip)

For pip-managed projects, uv is used to bootstrap a fresh Python installation. pip itself is bundled with CPython and needs no separate bootstrap.

```bash
# Bootstrap uv first (see uv section above) to get Python
# After uv bootstrap, SBOM_PYTHON is already set to the managed Python path

# Apply pip config from references/pip.conf (or private override)
PIP_CONFIG_DIR="${TEMP_DIR}/pip"
mkdir -p "${PIP_CONFIG_DIR}"
cp "${SOURCES_DIR}/pip.conf" "${PIP_CONFIG_DIR}/pip.conf"
export PIP_CONFIG_FILE="${PIP_CONFIG_DIR}/pip.conf"
```

**Cleanup:** removed automatically when `rm -rf "${TEMP_DIR}"` runs.

---

## CycloneDX SBOM tooling (always installed into SBOM_PYTHON)

After `SBOM_PYTHON` is set by the package-manager bootstrap above, install `cyclonedx-bom` into it. This keeps the SBOM tooling isolated from both the project environment and the user's system Python.

```bash
echo "Installing cyclonedx-bom into ${SBOM_PYTHON} ..."
"${SBOM_PYTHON}" -m pip install --quiet cyclonedx-bom

# Verify
"${SBOM_PYTHON}" -c "import cyclonedx; print('cyclonedx-bom', cyclonedx.__version__)"
```

If pip is not available on `SBOM_PYTHON` (unlikely with any modern CPython), fall back:

```bash
"${SBOM_PYTHON}" -m ensurepip --upgrade
"${SBOM_PYTHON}" -m pip install --quiet cyclonedx-bom
```

`SBOM_PYTHON` is the Python used for all SBOM generation (Strategy B/C scripts in `cyclonedx-generation.md`). It is never a project-environment Python.

---

## Creating project environments in TEMP_DIR

### pip / uv

```bash
# Create a venv inside TEMP_DIR — never inside PROJECT_ROOT
VENV_DIR="${TEMP_DIR}/venv"
"${SBOM_PYTHON}" -m venv "${VENV_DIR}"
# OR: uv venv "${VENV_DIR}" --python "${SBOM_PYTHON}"

# Install project dependencies into the TEMP venv
PIP_BIN="${VENV_DIR}/bin/pip"
export PIP_CONFIG_FILE="${PIP_CONFIG_DIR}/pip.conf"

# Detect manifest and install
if [ -f "${PROJECT_ROOT}/requirements.txt" ]; then
  "${PIP_BIN}" install -r "${PROJECT_ROOT}/requirements.txt" --quiet
elif [ -f "${PROJECT_ROOT}/pyproject.toml" ]; then
  "${PIP_BIN}" install "${PROJECT_ROOT}" --quiet   # installs project + deps
fi

# Query installed packages
"${VENV_DIR}/bin/pip" list --format json
```

### conda

```bash
# Each conda environment is created with --prefix inside TEMP_DIR.
# --prefix keeps everything in TEMP_DIR and does NOT add an entry to the
# user's conda environment registry (unlike --name).
CONDA_ENV_PREFIX="${TEMP_DIR}/conda-envs/$(basename "${YAML_PATH}" .yml)"
mkdir -p "${TEMP_DIR}/conda-envs"

conda env create \
  --file "${YAML_PATH}" \
  --prefix "${CONDA_ENV_PREFIX}" \
  --quiet

# Query
conda list --prefix "${CONDA_ENV_PREFIX}" --json

# Cleanup (after SBOM is written)
rm -rf "${CONDA_ENV_PREFIX}"
```

### pixi

pixi environments are managed inside the project directory (`.pixi/envs/`). The pixi binary itself lives in TEMP_DIR; only the project-local environment storage is in the project tree (as pixi intends).

```bash
# Use the TEMP pixi binary to list packages from the project environments
# This reads PROJECT_ROOT/pixi.toml but does not write outside the project
"${PIXI_INSTALL_DIR}/bin/pixi" list --json \
  --environment "${ENV_NAME}" \
  --manifest-path "${PROJECT_ROOT}/pixi.toml"
```

---

## Cleanup script (Step 7)

```bash
# Step 7a — remove any surviving conda temp environment prefixes
for prefix in "${CONDA_TEMP_PREFIXES[@]}"; do
  [ -d "${prefix}" ] && rm -rf "${prefix}" && echo "Removed conda env: ${prefix}"
done

# Step 7b — remove everything else (bootstrapped tools, venvs, caches)
if [ -n "${TEMP_DIR}" ] && [ -d "${TEMP_DIR}" ]; then
  echo "Removing TEMP_DIR: ${TEMP_DIR}"
  rm -rf "${TEMP_DIR}"
fi
```

`TEMP_DIR` contains all bootstrapped binaries, Python installations, caches, and conda environment prefixes. A single `rm -rf` removes everything. No user home directories or system paths are modified.

---

## Bootstrap decision table

| PKG_MANAGER | Tools always bootstrapped | Project env created in |
|---|---|---|
| `pip` | uv → Python 3.13 → cyclonedx-bom | `${TEMP_DIR}/venv` |
| `uv` | uv → Python 3.13 → cyclonedx-bom | `${TEMP_DIR}/venv` |
| `conda` | Miniforge → cyclonedx-bom in base Python | `${TEMP_DIR}/conda-envs/<name>` |
| `pixi` | pixi + uv → Python 3.13 → cyclonedx-bom | project-local `.pixi/envs/` (read-only via pixi list) |
