# CycloneDX 1.4 Specification Reference

Offline reference covering the subset of the CycloneDX 1.4 specification used by this skill. Authoritative source: https://cyclonedx.org/specification/overview/ and the JSON schema at https://cyclonedx.org/schema/bom-1.4.schema.json.

---

## Document structure

A CycloneDX 1.4 BOM (Bill of Materials) is a single JSON object or XML document. JSON is the canonical format used by this skill.

### Top-level fields

| Field | Type | Required | Description |
|---|---|---|---|
| `bomFormat` | string | **Yes** | Must be exactly `"CycloneDX"` |
| `specVersion` | string | **Yes** | Must be `"1.4"` for this version |
| `serialNumber` | string | No | Unique identifier for this BOM instance. Format: `urn:uuid:<uuid4>`. Strongly recommended — enables BOM deduplication and merging. |
| `version` | integer | No | BOM version number. Start at `1`; increment when a BOM for the same component is updated. |
| `metadata` | object | No | Metadata about the BOM itself — timestamp, tools, and the root component being described. |
| `components` | array | No | The list of components (libraries, frameworks, applications) included in the BOM. |
| `services` | array | No | External services; not used by this skill. |
| `externalReferences` | array | No | URLs to related resources; not used by this skill. |
| `dependencies` | array | No | Dependency graph — which components depend on which others. |
| `compositions` | array | No | Completeness assertions; not used by this skill. |
| `vulnerabilities` | array | No | Known vulnerabilities; not used by this skill. |

**Canonical key order used by this skill:**

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "serialNumber": "urn:uuid:...",
  "version": 1,
  "metadata": { ... },
  "components": [ ... ],
  "dependencies": [ ... ]
}
```

---

## `metadata` object

Describes the BOM itself and optionally the root component (the thing the BOM is about).

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | string (ISO 8601) | No | When the BOM was created. Example: `"2025-01-15T12:00:00+00:00"` |
| `tools` | array of tool objects | No | Software that created the BOM. See tool object below. |
| `authors` | array | No | Humans who created the BOM; not used by this skill. |
| `component` | component object | No | The root component being described (the application or library whose dependencies make up this BOM). When present, the `dependencies` section should list this component's `bom-ref` as the root node. |
| `manufacture` | object | No | The organization that manufactured the component; not used by this skill. |
| `supplier` | object | No | The organization that supplied the component; not used by this skill. |
| `licenses` | array | No | Licenses for the BOM itself; not used by this skill. |
| `properties` | array | No | Arbitrary name/value pairs; not used by this skill at the metadata level. |

### Tool object

```json
{
  "vendor": "generate-python-sbom-skill",
  "name": "generate-python-sbom",
  "version": "1.0.0"
}
```

| Field | Type | Required |
|---|---|---|
| `vendor` | string | No |
| `name` | string | No |
| `version` | string | No |

### Full metadata example

```json
"metadata": {
  "timestamp": "2025-01-15T12:00:00+00:00",
  "tools": [
    {
      "vendor": "generate-python-sbom-skill",
      "name": "generate-python-sbom",
      "version": "1.0.0"
    }
  ],
  "component": {
    "type": "application",
    "bom-ref": "pkg:pypi/my-project@1.0.0",
    "name": "my-project",
    "version": "1.0.0",
    "purl": "pkg:pypi/my-project@1.0.0",
    "properties": [
      { "name": "cdx:package-manager", "value": "pip" },
      { "name": "cdx:environment",     "value": "default" }
    ]
  }
}
```

---

## Component object

Used in both `components[]` (library/framework entries) and `metadata.component` (the root application). Fields are identical in both locations.

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string (enum) | **Yes** | Component type. See type values below. |
| `bom-ref` | string | No | Unique identifier for this component within the BOM. Used as the key in the `dependencies` graph. This skill uses the component's `purl` string as the `bom-ref` — making it stable and human-readable. |
| `name` | string | **Yes** | Component name as published (e.g., `"requests"`, `"Django"`). |
| `version` | string | No | Component version (e.g., `"2.31.0"`). Strongly recommended. |
| `purl` | string | No | Package URL. Strongly recommended — enables supply-chain tooling to identify the component unambiguously. See purl format section below. |
| `description` | string | No | Human-readable description. |
| `licenses` | array | No | SPDX license expressions or license objects. |
| `hashes` | array | No | Cryptographic hashes of the component artifact. |
| `externalReferences` | array | No | URLs to the component's homepage, VCS, issue tracker, etc. |
| `properties` | array of property objects | No | Arbitrary name/value pairs — the extension point used by this skill for `cdx:dependency:type`, `cdx:package-manager`, and `cdx:environment`. |
| `components` | array | No | Nested sub-components; not used by this skill. |

### Component type values

The `type` field is an enum. Values relevant to Python projects:

| Value | Meaning |
|---|---|
| `"library"` | A software library or package. Use for all entries in `components[]`. |
| `"application"` | An application. Use for `metadata.component` (the root project). |
| `"framework"` | A software framework; acceptable alternative to `"library"` for frameworks like Django. |
| `"container"` | A container image; not used by this skill. |
| `"device"` | Hardware; not used by this skill. |
| `"firmware"` | Firmware; not used by this skill. |

This skill uses `"library"` for all dependency components and `"application"` for the root metadata component.

### Property object

Properties are arbitrary name/value pairs attached to a component. The `properties` array is the standard CycloneDX extension point — it does not affect schema validation.

```json
{
  "name": "cdx:dependency:type",
  "value": "direct"
}
```

| Field | Type | Required |
|---|---|---|
| `name` | string | **Yes** |
| `value` | string | **Yes** |

**Properties used by this skill:**

| Property name | Values | Applies to |
|---|---|---|
| `cdx:dependency:type` | `"direct"` / `"transitive"` | Every component in `components[]` |
| `cdx:package-manager` | `"pixi"` / `"uv"` / `"pip"` / `"conda"` | Every component in `components[]` and `metadata.component` |
| `cdx:environment` | Environment name, e.g., `"default"`, `"py312"`, `"base"` | Every component in `components[]` and `metadata.component` |

The `cdx:` namespace prefix is a convention (not enforced by the schema) indicating these are CycloneDX-ecosystem extensions. The property names are chosen to be unambiguous and human-readable in SBOM tooling UIs.

### Full component example

```json
{
  "type": "library",
  "bom-ref": "pkg:pypi/requests@2.31.0",
  "name": "requests",
  "version": "2.31.0",
  "purl": "pkg:pypi/requests@2.31.0",
  "properties": [
    { "name": "cdx:dependency:type", "value": "direct" },
    { "name": "cdx:package-manager", "value": "pip" },
    { "name": "cdx:environment",     "value": "default" }
  ]
}
```

---

## Package URL (purl) format

Package URLs are the primary identity mechanism for components in supply-chain tooling. Defined by the PURL specification: https://github.com/package-url/purl-spec

### General format

```
pkg:<type>[/<namespace>]/<name>[@<version>][?<qualifiers>][#<subpath>]
```

| Segment | Required | Description |
|---|---|---|
| `pkg:` | Yes | Literal scheme prefix |
| `<type>` | Yes | Ecosystem type (e.g., `pypi`, `conda`, `npm`) |
| `/<namespace>` | Ecosystem-dependent | Channel or org namespace; see per-ecosystem rules below |
| `/<name>` | Yes | Package name, normalized per ecosystem rules |
| `@<version>` | No (strongly recommended) | Package version |
| `?<qualifiers>` | No | Key=value pairs for additional metadata; not used by this skill |
| `#<subpath>` | No | Path within the component; not used by this skill |

### PyPI packages

```
pkg:pypi/<name>@<version>
```

- Type: `pypi`
- No namespace
- Name: lowercase; hyphens only (normalize per PEP 503: `re.sub(r"[-_.]+", "-", name).lower()`)
- Version: exact version string from the package

Examples:
```
pkg:pypi/requests@2.31.0
pkg:pypi/django@5.2.15
pkg:pypi/django-rest-framework@3.17.1
```

Note: the canonical purl name for `djangorestframework` is `pkg:pypi/djangorestframework@3.17.1` — use the pip install name, not the import name. PEP 503 normalization applies to the install name.

### conda packages

```
pkg:conda/<channel>/<name>@<version>
```

- Type: `conda`
- Namespace: the channel (e.g., `conda-forge`, `defaults`, `bioconda`). If the channel is unknown, omit the namespace.
- Name: package name as listed by `conda list`
- Version: exact version string

Examples:
```
pkg:conda/conda-forge/numpy@1.26.4
pkg:conda/defaults/python@3.13.0
pkg:conda/numpy@1.26.4          ← when channel is unknown
```

**How to determine the channel** from `pixi list --json` or `conda list --json`:
- `pixi list`: use the `source` URL; extract the channel from `https://conda.anaconda.org/<channel>/...`
- `conda list`: use the `channel` field directly

PyPI packages installed inside a conda environment show `channel: "pypi"` — treat them as `pkg:pypi/<name>@<version>`, not `pkg:conda/...`.

### Name normalization for purl

PyPI enforces PEP 503 canonical names. Always normalize before building a purl:

```python
import re
def canonical(name: str) -> str:
    return re.sub(r"[-_.]+", "-", name).lower()
```

Examples:
- `Django` → `django`
- `Pillow` → `pillow`
- `scikit_learn` → `scikit-learn`
- `typing.extensions` → `typing-extensions`

conda package names are used as-is (no normalization needed — conda names do not follow PEP 503).

---

## `dependencies` array

The dependency graph. Each entry describes what one component directly depends on.

```json
"dependencies": [
  {
    "ref": "pkg:pypi/my-project@1.0.0",
    "dependsOn": [
      "pkg:pypi/requests@2.31.0",
      "pkg:pypi/click@8.1.7"
    ]
  }
]
```

| Field | Type | Required | Description |
|---|---|---|---|
| `ref` | string | **Yes** | The `bom-ref` of the component that has dependencies. Must match a `bom-ref` defined in `components[]` or `metadata.component`. |
| `dependsOn` | array of strings | No | The `bom-ref` values of the components this component directly depends on. Empty array means the component has no known dependencies. Omitting `dependsOn` means the relationship is unknown (different from "no dependencies"). |

**How this skill uses the dependency graph:**

The skill records only the root→direct-deps relationship:

```json
"dependencies": [
  {
    "ref": "pkg:pypi/my-project@1.0.0",
    "dependsOn": [
      "pkg:pypi/requests@2.31.0",
      "pkg:pypi/click@8.1.7"
    ]
  }
]
```

Transitive relationships (e.g., `requests → certifi`) are not recorded because pip/conda/pixi do not expose this graph in their JSON output without additional tooling (e.g., `pixi tree`, `pip-tree`). The `cdx:dependency:type` property on each component records whether it is direct or transitive.

A complete dependency graph (every component listed with its own `dependsOn`) is preferred by SBOM analysis tools but is not required by the 1.4 schema.

---

## `bom-ref` rules

`bom-ref` is a free-form string that must be:

1. **Unique** within the BOM — no two components may share the same `bom-ref`
2. **Consistent** — any `ref` or `dependsOn` value in `dependencies[]` must match a `bom-ref` defined on a component

This skill uses the component's `purl` string as the `bom-ref`. This has several advantages:
- Deterministic across runs (same package → same `bom-ref`)
- Human-readable
- Unique by definition (purl uniquely identifies a component version)
- Avoids the random IDs generated by some libraries (e.g., `BomRef.8272730...`)

---

## `serialNumber` format

`serialNumber` uniquely identifies this specific BOM document. Two BOM documents that describe the same component at the same point in time should have different `serialNumber` values.

Format: `urn:uuid:<uuid4>`

```
urn:uuid:d8b8cc41-ad4d-44fa-9ad3-186c7a942865
```

Generate with Python: `f"urn:uuid:{uuid.uuid4()}"`

The `version` field (integer, starts at 1) is used together with `serialNumber` to track updates to the same BOM document. If you regenerate the BOM for the same component with a new `serialNumber`, start `version` at 1. If you update a BOM in-place and keep the same `serialNumber`, increment `version`.

---

## JSON vs XML

CycloneDX 1.4 is defined in both JSON and XML. This skill uses JSON by default.

**JSON:** straightforward key/value structure as documented in this file.

**XML:** wraps the same data in XML elements under the `http://cyclonedx.org/schema/bom/1.4` namespace:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bom xmlns="http://cyclonedx.org/schema/bom/1.4"
     serialNumber="urn:uuid:d8b8cc41-ad4d-44fa-9ad3-186c7a942865"
     version="1">
  <metadata>
    <timestamp>2025-01-15T12:00:00Z</timestamp>
    <tools>
      <tool>
        <vendor>generate-python-sbom-skill</vendor>
        <name>generate-python-sbom</name>
        <version>1.0.0</version>
      </tool>
    </tools>
    <component type="application">
      <name>my-project</name>
      <version>1.0.0</version>
      <purl>pkg:pypi/my-project@1.0.0</purl>
    </component>
  </metadata>
  <components>
    <component type="library">
      <name>requests</name>
      <version>2.31.0</version>
      <purl>pkg:pypi/requests@2.31.0</purl>
      <properties>
        <property name="cdx:dependency:type">direct</property>
        <property name="cdx:package-manager">pip</property>
        <property name="cdx:environment">default</property>
      </properties>
    </component>
  </components>
</bom>
```

Key XML differences from JSON:
- `bom-ref` becomes the `bom-ref` XML attribute on `<component>`: `<component type="library" bom-ref="pkg:pypi/requests@2.31.0">`
- `bomFormat` and `specVersion` are implied by the namespace URI — no separate fields
- `serialNumber` and `version` are attributes on `<bom>`, not child elements
- Property values are element text, not a JSON object: `<property name="...">value</property>`

---

## Validation

The official schemas are published at:
- JSON: `https://cyclonedx.org/schema/bom-1.4.schema.json`
- XML: `https://cyclonedx.org/schema/bom-1.4.xsd`

Runtime validation requires the `cyclonedx-cli` Go binary (separate from the Python `cyclonedx-bom` package):

```bash
# Install (macOS)
brew install cyclonedx/cyclonedx/cyclonedx-cli

# Or download from https://github.com/CycloneDX/cyclonedx-cli/releases

# Validate
cyclonedx validate --input-format json --input-version v1_4 --input-file sbom.cyclonedx.json
cyclonedx validate --input-format xml  --input-version v1_4 --input-file sbom.cyclonedx.xml
```

Exit 0 = valid. Non-zero exit with output = schema violation.

### Common validation errors

| Error | Cause | Fix |
|---|---|---|
| `bomFormat` is required | Field missing or misspelled | Add `"bomFormat": "CycloneDX"` |
| `specVersion` is required | Field missing | Add `"specVersion": "1.4"` |
| `type` is required on component | Component missing type | Add `"type": "library"` or `"type": "application"` |
| `name` is required on component | Component missing name | Add `"name": "<package-name>"` |
| `serialNumber` does not match pattern | Not in `urn:uuid:<uuid4>` format | Use `f"urn:uuid:{uuid.uuid4()}"` |
| `ref` in dependencies not found | `bom-ref` in `dependencies[].ref` doesn't match any component | Ensure `bom-ref` values in components match exactly what is referenced in `dependencies` |
| `version` must be integer | Top-level `version` field set to a string | Set `"version": 1` (not `"version": "1"`) |

---

## Minimum valid BOM

The smallest JSON that passes CycloneDX 1.4 schema validation:

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4"
}
```

In practice, a useful BOM must have at least `serialNumber`, `metadata.timestamp`, and a non-empty `components` array.

---

## Schema evolution — 1.4 vs 1.5 / 1.6

This skill targets 1.4. Notable additions in later versions (for context if upgrading):

| Feature | Added in |
|---|---|
| `vulnerabilities` top-level array | 1.4 |
| `compositions` completeness assertions | 1.4 |
| `formulation` (build reproducibility) | 1.5 |
| `annotations` (notes on components) | 1.5 |
| `crypto-assets` component type | 1.6 |
| Machine-learning model assets | 1.6 |

All fields documented in this file are valid in 1.4, 1.5, and 1.6 without modification — the schema is additive between versions.
