# CycloneDX Generation Reference

Python script templates for SBOM generation. The skill selects among three strategies:

- **Strategy A** — `cyclonedx-py` CLI; only viable for pip/uv with an activated venv
- **Strategy B** — programmatic via `cyclonedx-python-lib`; requires the library in `GLOBAL_PYTHON`
- **Strategy C** — stdlib-only; no third-party dependency; always available; used for multi-env pixi/conda

A single entry-point script below handles both Strategy B and C via the `USE_STDLIB_ONLY` flag.

---

## Strategy B / C — Unified generation script

Save as a temporary file (e.g., `/tmp/generate_sbom.py`) and run with `${GLOBAL_PYTHON} /tmp/generate_sbom.py`.

Substitute these template variables before writing:

| Variable | Source |
|---|---|
| `${INSTALLED_PACKAGES_JSON}` | Raw JSON from `pixi list --json`, `conda list --json`, `pip list --format json`, etc. |
| `${DIRECT_DEP_NAMES_JSON}` | JSON array of canonical direct dependency names from Step 2 |
| `${FORMAT}` | `json` or `xml` |
| `${OUTPUT}` | Absolute output file path (already includes env-name suffix if multi-env) |
| `${PKG_MANAGER}` | `pixi`, `uv`, `conda`, or `pip` |
| `${ENV_NAME}` | Current environment name (e.g., `default`, `py312`, `myenv`, `base`) |
| `${PROJECT_NAME}` | From manifest `name` field; empty string if not found |
| `${PROJECT_VERSION}` | From manifest; `0.0.0` if not found |
| `${USE_STDLIB_ONLY}` | `true` to skip cyclonedx-python-lib and generate JSON directly |

```python
"""CycloneDX SBOM generator — supports Strategy B (cyclonedx-python-lib) and C (stdlib-only)."""

from __future__ import annotations

import json
import re
import uuid
from datetime import datetime, timezone
from pathlib import Path

# --- Configuration (substituted by the skill before execution) ---
INSTALLED_PACKAGES_JSON: str = """${INSTALLED_PACKAGES_JSON}"""
DIRECT_DEP_NAMES_JSON: str = """${DIRECT_DEP_NAMES_JSON}"""
FORMAT: str = "${FORMAT}"
OUTPUT: str = "${OUTPUT}"
PKG_MANAGER: str = "${PKG_MANAGER}"
ENV_NAME: str = "${ENV_NAME}"
PROJECT_NAME: str = "${PROJECT_NAME}"
PROJECT_VERSION: str = "${PROJECT_VERSION}"
USE_STDLIB_ONLY: bool = "${USE_STDLIB_ONLY}".lower() == "true"


def canonical(name: str) -> str:
    return re.sub(r"[-_.]+", "-", name).lower()


def build_purl_str(pkg: dict) -> str:
    name = canonical(pkg["name"])
    raw_version = pkg.get("version")
    # Editable self-installs (e.g. pixi pypi-dependencies with path=".") report
    # None for version; substitute the project version so the purl is valid.
    if not raw_version and canonical(pkg["name"]) == canonical(PROJECT_NAME):
        raw_version = PROJECT_VERSION
    version = raw_version or ""
    kind = pkg.get("kind", "")
    channel = pkg.get("channel", "")
    ver_suffix = f"@{version}" if version else ""

    if PKG_MANAGER in ("pip", "uv") or kind == "pypi" or channel == "pypi":
        return f"pkg:pypi/{name}{ver_suffix}"
    if PKG_MANAGER in ("pixi", "conda"):
        ns = channel if channel and channel != "pypi" else "conda-forge"
        return f"pkg:conda/{ns}/{name}{ver_suffix}"
    return f"pkg:generic/{name}{ver_suffix}"


# ── Strategy C: stdlib-only JSON generation ──────────────────────────────────

def generate_stdlib(installed: list[dict], direct_names: set[str]) -> None:
    pname = PROJECT_NAME if PROJECT_NAME and "${" not in PROJECT_NAME else ""
    pver = PROJECT_VERSION if PROJECT_VERSION and "${" not in PROJECT_VERSION else "0.0.0"
    root_purl = f"pkg:pypi/{canonical(pname)}@{pver}" if pname else ""

    components = []
    for pkg in installed:
        purl = build_purl_str(pkg)
        is_direct = canonical(pkg["name"]) in direct_names
        raw_ver = pkg.get("version")
        if not raw_ver and canonical(pkg["name"]) == canonical(pname):
            raw_ver = pver
        components.append({
            "type": "library",
            "bom-ref": purl,
            "name": pkg["name"],
            "version": raw_ver or "",
            "purl": purl,
            "properties": [
                {"name": "cdx:dependency:type", "value": "direct" if is_direct else "transitive"},
                {"name": "cdx:package-manager", "value": PKG_MANAGER},
                {"name": "cdx:environment", "value": ENV_NAME},
            ],
        })

    direct_purls = [c["bom-ref"] for c in components if any(p["value"] == "direct" for p in c["properties"] if p["name"] == "cdx:dependency:type")]

    # Build metadata block — component key only present when project name is known
    metadata: dict = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "tools": [{"vendor": "generate-python-sbom", "name": "generate-python-sbom", "version": "1.0.0"}],
    }
    if pname and root_purl:
        metadata["component"] = {
            "type": "application",
            "bom-ref": root_purl,
            "name": pname,
            "version": pver,
            "purl": root_purl,
            "properties": [
                {"name": "cdx:package-manager", "value": PKG_MANAGER},
                {"name": "cdx:environment", "value": ENV_NAME},
            ],
        }

    # Canonical key order — all consumers must see this same structure regardless of strategy
    bom: dict = {
        "bomFormat": "CycloneDX",
        "specVersion": "1.4",
        "serialNumber": f"urn:uuid:{uuid.uuid4()}",
        "version": 1,
        "metadata": metadata,
        "components": components,
    }
    if pname and root_purl:
        bom["dependencies"] = [{"ref": root_purl, "dependsOn": direct_purls}]

    output_path = Path(OUTPUT)
    output_path.parent.mkdir(parents=True, exist_ok=True)

    if FORMAT.lower() == "json":
        output_path.write_text(json.dumps(bom, indent=2), encoding="utf-8")
    else:
        # Minimal CycloneDX 1.4 XML serialization
        lines = [
            '<?xml version="1.0" encoding="UTF-8"?>',
            '<bom xmlns="http://cyclonedx.org/schema/bom/1.4"',
            f'     serialNumber="{bom["serialNumber"]}" version="1">',
            "  <components>",
        ]
        for c in components:
            dep_type = next((p["value"] for p in c["properties"] if p["name"] == "cdx:dependency:type"), "transitive")
            lines += [
                f'    <component type="library">',
                f'      <name>{c["name"]}</name>',
                f'      <version>{c["version"]}</version>',
                f'      <purl>{c["purl"]}</purl>',
                f'      <properties>',
                f'        <property name="cdx:dependency:type">{dep_type}</property>',
                f'        <property name="cdx:environment">{ENV_NAME}</property>',
                f'      </properties>',
                f'    </component>',
            ]
        lines += ["  </components>", "</bom>"]
        output_path.write_text("\n".join(lines), encoding="utf-8")

    total = len(components)
    direct_count = sum(1 for c in components if any(p["value"] == "direct" for p in c["properties"] if p["name"] == "cdx:dependency:type"))
    print(f"SBOM written: {output_path.resolve()}")
    print(f"Schema: CycloneDX 1.4 ({FORMAT.upper()}) | Environment: {ENV_NAME}")
    print(f"Components: {total} ({direct_count} direct, {total - direct_count} transitive)")


# ── Strategy B: cyclonedx-python-lib ─────────────────────────────────────────

def generate_with_lib(installed: list[dict], direct_names: set[str]) -> None:
    try:
        from cyclonedx.model.bom import Bom
        from cyclonedx.model.component import Component, ComponentType
        from cyclonedx.model import Property
        from cyclonedx.output import make_outputter
        from cyclonedx.schema import OutputFormat, SchemaVersion
        from packageurl import PackageURL
    except ImportError as exc:
        print(f"cyclonedx-python-lib not importable ({exc}); falling back to stdlib generation")
        generate_stdlib(installed, direct_names)
        return

    def to_purl(pkg: dict) -> PackageURL:
        purl_str = build_purl_str(pkg)
        # PackageURL.from_string parses pkg:type/ns/name@ver
        return PackageURL.from_string(purl_str)

    bom = Bom()
    bom.serial_number = uuid.uuid4()

    for pkg in installed:
        is_direct = canonical(pkg["name"]) in direct_names
        props = [
            Property(name="cdx:dependency:type", value="direct" if is_direct else "transitive"),
            Property(name="cdx:package-manager", value=PKG_MANAGER),
            Property(name="cdx:environment", value=ENV_NAME),
        ]
        component = Component(
            type=ComponentType.LIBRARY,   # v8+: 'type'; v7.x used 'component_type'
            name=pkg["name"],
            version=pkg.get("version", ""),
            purl=to_purl(pkg),
            properties=props,
        )
        bom.components.add(component)

    pname = PROJECT_NAME if PROJECT_NAME and "${" not in PROJECT_NAME else ""
    if pname:
        pver = PROJECT_VERSION if PROJECT_VERSION and "${" not in PROJECT_VERSION else "0.0.0"
        bom.metadata.component = Component(
            type=ComponentType.APPLICATION,   # v8+: 'type'; v7.x used 'component_type'
            name=pname,
            version=pver,
        )

    output_format = OutputFormat.JSON if FORMAT.lower() == "json" else OutputFormat.XML
    outputter = make_outputter(bom, output_format, SchemaVersion.V1_4)

    output_path = Path(OUTPUT)
    output_path.parent.mkdir(parents=True, exist_ok=True)
    output_path.write_text(outputter.output_as_string(indent=2), encoding="utf-8")

    total = len(installed)
    direct_count = sum(1 for p in installed if canonical(p["name"]) in direct_names)
    print(f"SBOM written: {output_path.resolve()}")
    print(f"Schema: CycloneDX 1.4 ({FORMAT.upper()}) | Environment: {ENV_NAME}")
    print(f"Components: {total} ({direct_count} direct, {total - direct_count} transitive)")


# ── Entry point ───────────────────────────────────────────────────────────────

def main() -> None:
    installed: list[dict] = json.loads(INSTALLED_PACKAGES_JSON)
    direct_names: set[str] = {canonical(n) for n in json.loads(DIRECT_DEP_NAMES_JSON)}

    if FORMAT.lower() == "json":
        # JSON always uses generate_stdlib regardless of library availability.
        # This guarantees identical output structure across all strategies —
        # the library's JSON serializer produces different key ordering, random
        # bom-ref values, and missing metadata fields compared to our canonical format.
        generate_stdlib(installed, direct_names)
    elif USE_STDLIB_ONLY:
        generate_stdlib(installed, direct_names)
    else:
        # XML only: use the library for correct namespace/entity handling.
        generate_with_lib(installed, direct_names)


if __name__ == "__main__":
    main()
```

---

## Strategy A patch — Annotate direct deps in cyclonedx-py CLI output

After `cyclonedx-py environment` generates the SBOM, run this to add `cdx:dependency:type` and `cdx:environment` properties. JSON only; XML requires namespace-aware patching.

Substitute `${OUTPUT}`, `${DIRECT_DEP_NAMES_JSON}`, and `${ENV_NAME}` before running.

```python
"""Patch a CycloneDX JSON SBOM: add cdx:dependency:type and cdx:environment properties."""

from __future__ import annotations

import json
import re
from pathlib import Path

OUTPUT = "${OUTPUT}"
DIRECT_DEP_NAMES_JSON = """${DIRECT_DEP_NAMES_JSON}"""
ENV_NAME = "${ENV_NAME}"


def canonical(name: str) -> str:
    return re.sub(r"[-_.]+", "-", name).lower()


direct_names: set[str] = {canonical(n) for n in json.loads(DIRECT_DEP_NAMES_JSON)}
path = Path(OUTPUT)

with path.open() as f:
    bom: dict = json.load(f)

for component in bom.get("components", []):
    dep_type = "direct" if canonical(component.get("name", "")) in direct_names else "transitive"
    props = [p for p in component.get("properties", []) if p.get("name") not in ("cdx:dependency:type", "cdx:environment")]
    props.append({"name": "cdx:dependency:type", "value": dep_type})
    props.append({"name": "cdx:environment", "value": ENV_NAME})
    component["properties"] = props

path.write_text(json.dumps(bom, indent=2), encoding="utf-8")
print(f"Patched {len(bom.get('components', []))} components in {path}")
```

---

## CycloneDX 1.4 JSON structure reference

Minimal valid SBOM skeleton:

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "serialNumber": "urn:uuid:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "version": 1,
  "metadata": {
    "timestamp": "2024-01-15T12:00:00Z",
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
        { "name": "cdx:environment", "value": "default" }
      ]
    }
  ],
  "dependencies": [
    { "ref": "pkg:pypi/my-project@1.0.0", "dependsOn": ["pkg:pypi/requests@2.31.0"] }
  ]
}
```

Key fields:
- `bomFormat` — must be `"CycloneDX"`
- `specVersion` — `"1.4"`
- `serialNumber` — `urn:uuid:<uuid4>`
- `components[].purl` — Package URL (required for supply-chain tooling)
- `cdx:dependency:type` property — `direct` or `transitive`
- `cdx:environment` property — environment name; enables diff-by-env in SBOM tools

---

## Validation

`cyclonedx-py` (from the `cyclonedx-bom` PyPI package) is a generation tool only — it has no `validate` subcommand. Schema validation requires the separate `cyclonedx-cli` Go binary:

```bash
# Install cyclonedx-cli (Go binary — not a Python package)
brew install cyclonedx/cyclonedx/cyclonedx-cli   # macOS
# or download from https://github.com/CycloneDX/cyclonedx-cli/releases

cyclonedx validate --input-format json --input-version v1_4 --input-file sbom.cyclonedx.json
cyclonedx validate --input-format xml  --input-version v1_4 --input-file sbom.cyclonedx.xml
```

Exit 0 = valid. Any output with non-zero exit indicates a schema violation.

---

## cyclonedx-python-lib version compatibility

| Library version | Supported schema | `make_outputter` available | `Component` kwarg |
|---|---|---|---|
| `>=8.0` (incl. 11.x) | 1.4, 1.5, 1.6 | Yes | `type=` |
| `7.x` | 1.4, 1.5, 1.6 | Yes | `component_type=` |
| `4.x–6.x` | 1.4 | No — use direct class: `from cyclonedx.output.json import JsonV1Dot4` | `component_type=` |

**Breaking change at v8.0:** `Component.__init__` renamed `component_type` to `type`. The script above uses `type=` (v8+ spelling). If running against v7.x, change `type=ComponentType.LIBRARY` to `component_type=ComponentType.LIBRARY`. The script falls back to `generate_stdlib` on `ImportError`, so if the library is absent it still works.
