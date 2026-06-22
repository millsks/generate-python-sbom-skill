# Installation Sources

This file is the single place to change when moving between environments (public internet, private mirror, corporate artifact proxy).

Default values below point to open-source community sources.
Replace URLs and version pins here for air-gapped, corporate, or private deployments.

---

## Python (CPython)

| Field | Value |
|---|---|
| Distributor | Astral — builds from official CPython source via python-build-standalone |
| Install mechanism | `uv python install <version>` |
| Standalone release source | https://github.com/astral-sh/python-build-standalone/releases |
| Target version | `3.13` |
| Private replacement | Set `UV_PYTHON_DOWNLOADS=never` and point `UV_PYTHON_INSTALL_DIR` to a pre-cached directory |

---

## uv

| Field | Value |
|---|---|
| Installer script | `https://astral.sh/uv/install.sh` |
| Version | `latest` (pin a tag for reproducibility, e.g. `https://github.com/astral-sh/uv/releases/download/0.6.14/uv-installer.sh`) |
| Release index | https://github.com/astral-sh/uv/releases |
| Install env var | `UV_INSTALL_DIR` controls destination directory |
| Private replacement | Replace the installer URL with an internal download path; set `UV_INDEX_URL` and `UV_EXTRA_INDEX_URL` for package resolution |

---

## Pixi

| Field | Value |
|---|---|
| Installer script | `https://pixi.sh/install.sh` |
| Version | `latest` (pin via `PIXI_VERSION` env var, e.g. `PIXI_VERSION=0.46.0`) |
| Release index | https://github.com/prefix-dev/pixi/releases |
| Install env var | `PIXI_HOME` controls destination directory |
| Private replacement | Replace the installer URL with an internal download path; set `PIXI_DEFAULT_CHANNELS` and `default-channels` in `pixi-config.toml` |

---

## Miniforge (conda)

| Field | Value |
|---|---|
| Base URL | `https://github.com/conda-forge/miniforge/releases/latest/download/` |
| macOS ARM64 | `Miniforge3-MacOSX-arm64.sh` |
| macOS x86_64 | `Miniforge3-MacOSX-x86_64.sh` |
| Linux aarch64 | `Miniforge3-Linux-aarch64.sh` |
| Linux x86_64 | `Miniforge3-Linux-x86_64.sh` |
| Windows x86_64 | `Miniforge3-Windows-x86_64.exe` |
| Version | `latest` (pin by replacing `latest` with a tag, e.g. `26.3.2-3`) |
| Release index | https://github.com/conda-forge/miniforge/releases |
| Private replacement | Replace the installer URL with an internal Artifactory/Nexus path; set `channel_alias` and `channels` in `condarc` to point to internal mirrors |

> **macOS vs Linux arch naming:** macOS installers use `arm64`; Linux installers use `aarch64`. These are the same hardware but use different names in Miniforge asset filenames.

---

## CycloneDX SBOM tooling

| Field | Value |
|---|---|
| Package name (PyPI) | `cyclonedx-bom` |
| PyPI URL | https://pypi.org/project/cyclonedx-bom/ |
| Version | `latest` |
| Install command | `pip install cyclonedx-bom` |
| Private replacement | Add an internal PyPI mirror as extra index or replace index URL in `pip.conf` |

---

## How to adapt for a private / corporate environment

1. Copy this file and the per-manager config files (`pip.conf`, `condarc`, `uv-config.toml`, `pixi-config.toml`) into a private repository or team configuration directory.
2. Update URLs to point to internal artifact repositories (Artifactory, Nexus, Devpi, Conda Mirror, etc.).
3. Pin version numbers to approved releases if the corporate environment requires change-controlled tooling.
4. Reference the private copies by setting the env var `SBOM_SKILL_SOURCES_DIR` to the directory containing the overrides. The skill will prefer files in that directory over the built-in defaults.
