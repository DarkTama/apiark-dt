# 1. Insomnia v5 Format Support

Date: 2026-03-30

## Status

Accepted

## Context

ApiArk previously only supported importing and exporting Insomnia collections in the legacy v4 format (`_type: export`, `__export_format: 4`, flat `resources` array). Modern Kong Insomnia versions export collections as v5 YAML documents by default (`type: collection.insomnia.rest/5.0`, `schema_version: "5.1"`). Users attempting to import modern Insomnia collections encounter format detection failures or errors indicating missing `resources`.

Additionally:
- Insomnia v5 uses hierarchical trees with `children` instead of a flat resource list with parent IDs.
- Insomnia v5 references environment variables with a `_.` prefix (`{{ _.var }}`).
- Insomnia v5 encapsulates sub-environments inside a root environment with inheritance.
- Insomnia v5 stores pre-request scripts and response tests under `scripts.preRequest` and `scripts.afterResponse`.

## Decision

1. **Dual Import Support**: Auto-detect both Insomnia v4 (JSON/YAML with `resources` or `__export_format`) and Insomnia v5 (YAML/JSON with `type: *.insomnia.rest/5.*` or `collection.insomnia.rest/*`).
2. **Variable Normalization**: Strip the `_.` prefix during import (`{{ _.foo }}` -> `{{ foo }}`) and restore it (`{{ foo }}` -> `{{ _.foo }}`) when exporting to v5 YAML.
3. **Environment Inheritance Flattening**: Merge base environment variables into each sub-environment upon import, writing standalone `.apiark/environments/<name>.yaml` files along with `Base.yaml`.
4. **Bidirectional Scripts**: Map Insomnia v5 `scripts.preRequest` and `scripts.afterResponse` to ApiArk `pre_request_script` and `post_response_script`.
5. **Switchable Exporter**: Support both v5 YAML export (default under `--format insomnia` and UI) and legacy v4 JSON export (`--format insomnia-v4`).
6. **Unified Parity**: Implement detection, import, and export across both Tauri desktop backend (`apps/desktop/src-tauri`) and CLI (`apps/cli`).

## Consequences

- Users can seamlessly import legacy v4 and modern v5 Insomnia exports via GUI and CLI without manual conversion.
- Imported environment variables work immediately with ApiArk's interpolation engine without syntax errors.
- Both legacy v4 JSON and modern v5 YAML export targets are preserved for interoperability.
