# Examples

Sample SBOM outputs produced by the generate-python-sbom skill. The commands shown below use Claude Code syntax; see [Invoking from different tools](../README.md#invoking-from-different-tools) for equivalent prompts in GitHub Copilot Agent, Devin, and other LLMs.

## sbom.cyclonedx.default.json

- **Source project:** `analyze-pipeline` v0.3.1 — a pixi/conda-forge project with a `default` and a `dev` environment
- **Command:** `/generate-python-sbom --path /projects/analyze-pipeline --env default`
- **Package manager:** pixi
- **Components:** 9 (4 direct, 5 transitive)
- **Notes:** The `.default` infix in the filename is added automatically when the project has multiple environments. Conda-forge components use `pkg:conda/conda-forge/...` purls; the root project itself uses `pkg:pypi/...` because it is the Python package being described, not one of its conda-installed deps.

## sbom.cyclonedx.xml

- **Source project:** `web-scraper` v1.2.0 — a plain pip project with a single `requirements.txt`
- **Command:** `/generate-python-sbom --path /projects/web-scraper --format xml`
- **Package manager:** pip
- **Components:** 8 (3 direct, 5 transitive)
- **Notes:** Single environment → no env infix in the filename. XML output is generated via Strategy B (cyclonedx-python-lib) for correct namespace and entity handling; `bom-ref` appears as an XML attribute on each `<component>` element.

## Validating these files

```bash
# Install the CycloneDX CLI (Go binary — separate from the Python cyclonedx-bom package)
brew install cyclonedx/cyclonedx/cyclonedx-cli

cyclonedx validate --input-format json --input-version v1_4 --input-file sbom.cyclonedx.default.json
cyclonedx validate --input-format xml  --input-version v1_4 --input-file sbom.cyclonedx.xml
```
