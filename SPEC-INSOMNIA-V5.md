# Technical Specification: Insomnia v5 Format Support

- **Document:** SPEC-INSOMNIA-V5
- **Status:** Draft / Ready for Implementation
- **Target:** `apps/desktop/src-tauri`, `apps/cli`, `packages/types`, `apps/desktop/src`

---

## 1. Objective

Provide first-class support for Kong Insomnia v5 export format across ApiArk Desktop and CLI:
1. **Auto-detection**: Detect both legacy v4 (`__export_format: 4`, flat `resources` array) and modern v5 (`type: collection.insomnia.rest/5.0`, `schema_version: "5.1"`, hierarchical `collection` tree) exports in both YAML and JSON.
2. **Import**: Fully parse v5 collections, folders, requests, headers, query parameters, bodies (JSON, form-data, urlencoded, GraphQL), auth, scripts (`scripts.preRequest`, `scripts.afterResponse`), and environments with inheritance.
3. **Variable Normalization**: Convert Insomnia `{{ _.var_name }}` syntax into ApiArk standard `{{ var_name }}` syntax during import, and reverse during v5 export.
4. **Dual Export**: Export to modern Insomnia v5 YAML by default (`--format insomnia`), with backwards-compatible Insomnia v4 JSON export (`--format insomnia-v4`).
5. **UI & CLI Parity**: Update CLI commands and desktop dialogs, formats, and locale labels.

---

## 2. Format Specifications

### 2.1 Insomnia v5 Schema (`collection.insomnia.rest/5.0`)

```yaml
type: collection.insomnia.rest/5.0
schema_version: "5.1"
name: <Collection Name>
meta:
  id: wrk_<id>
  created: <epoch_ms>
  modified: <epoch_ms>
  description: <string>
collection:
  - name: <Folder Name>
    meta:
      id: fld_<id>
      created: <epoch_ms>
      modified: <epoch_ms>
      sortKey: <number>
    children:
      - url: "{{ _.base_url }}/endpoint"
        name: <Request Name>
        meta:
          id: req_<id>
          created: <epoch_ms>
          modified: <epoch_ms>
        method: POST
        headers:
          - name: Content-Type
            value: application/json
            disabled: false
        parameters:
          - name: filter
            value: active
            disabled: false
        body:
          mimeType: application/json
          text: '{"key": "value"}'
        authentication:
          type: bearer
          token: "{{ _.token }}"
          disabled: false
        scripts:
          preRequest: "insomnia.test(...)"
          afterResponse: "insomnia.test(...)"
environments:
  name: Base Environment
  meta:
    id: env_<id>
  data:
    base_url: https://api.example.com
  subEnvironments:
    - name: Dev
      meta:
        id: env_<id>
      data:
        base_url: https://dev.example.com
```

---

## 3. Architecture & Component Changes

### 3.1 Detection Logic

#### Desktop (`apps/desktop/src-tauri/src/commands/import_export.rs`)
- File content inspected for:
  - `insomnia.rest/5.` or `collection.insomnia.rest`
  - `schema_version:` with `collection:`
  - Legacy `resources:` + `_type:` / `__export_format`
- Returns `"insomnia"` for both versions to dispatch to unified parser.

#### CLI (`apps/cli/src/import_export.rs`)
- In `detect_format()`:
  - Match JSON/YAML containing `insomnia.rest/` or `resources` or `__export_format`.
  - Return `"insomnia"`.

---

### 3.2 Importer Design (`apps/desktop/src-tauri/src/importer/insomnia.rs`)

1. **Parser Dispatch**:
   - Deserialize root into generic `serde_json::Value` (handling JSON or YAML).
   - If `root.get("type").and_then(|t| t.as_str()).contains("insomnia.rest/5")` or `root.get("collection").is_some()`:
     - Route to `parse_insomnia_v5(&root)`.
   - Else if `root.get("resources").is_some()`:
     - Route to existing `parse_insomnia_v4(&root)`.
   - Else: return descriptive error.

2. **V5 Collection Traversal**:
   - Recursively traverse `collection` array:
     - If item contains `children` -> parse as `ImportItem::Folder { name, items }`.
     - If item contains `url` or `method` -> parse as `ImportItem::Request`.

3. **Variable Normalization**:
   - Implement `strip_insomnia_env_prefix(input: &str) -> String`:
     - Replaces `\{\{\s*_\.([a-zA-Z0-9_\-\.]+)\s*\}\}` with `{{ $1 }}`.
     - Also trims non-breaking spaces (`\u{00a0}`) often introduced in copy-pasted Insomnia URLs.

4. **Request Mapping**:
   - `url`: apply `strip_insomnia_env_prefix` and trim whitespace.
   - `headers`: iterate array; skip if `disabled == true`; apply variable normalization to values.
   - `parameters`: iterate array; skip if `disabled == true`; insert into `params: HashMap<String, String>`; apply variable normalization.
   - `body`:
     - `application/json`, `application/xml`, `text/*`: body type `json`/`xml`/`raw`, content = normalized `text`.
     - `application/graphql`: body type `graphql`, content = `text`.
     - `multipart/form-data`: inspect `params` array:
       - Format as JSON array of `{ "key": name, "value": value_or_fileName, "type": "file"|"text" }`.
     - `application/x-www-form-urlencoded`: inspect `params` array:
       - Format as URL-encoded string (`key=value&...`).
   - `authentication`:
     - If `disabled == true` -> return `None`.
     - `bearer` -> `AuthConfig::Bearer { token: strip_prefix(token) }`.
     - `basic` -> `AuthConfig::Basic { username, password }`.
     - `apikey` -> `AuthConfig::ApiKey`.
     - Other -> log warning in `warnings`.
   - `scripts`:
     - `scripts.preRequest` -> `pre_request_script`.
     - `scripts.afterResponse` -> `post_response_script`.

5. **Environment Resolution with Inheritance**:
   - Extract `environments.data` (Base Environment variables).
   - Extract `environments.subEnvironments` array.
   - For each sub-environment:
     - Clone base variables map.
     - Overlay sub-environment's `data` map (child values take precedence).
     - Push `ImportEnvironment { name: sub_env.name, variables: merged_map }`.
   - If base variables map is non-empty, also push `ImportEnvironment { name: "Base", variables: base_map }`.

---

### 3.3 CLI Importer Design (`apps/cli/src/import_export.rs`)

Mirror desktop parser:
- Implement `import_insomnia` with v4/v5 detection.
- Add `import_insomnia_v5` creating folder directories and request `.yaml` files.
- Merge and write `.apiark/environments/<env>.yaml` files with inheritance.

---

### 3.4 Exporter Design (`apps/desktop/src-tauri/src/exporter/insomnia.rs`)

1. **Format Routing**:
   - Add `export_to_insomnia_v5(collection_path: &Path) -> Result<String, String>`.
   - Retain `export_to_insomnia_v4(collection_path: &Path) -> Result<String, String>` (legacy).
   - In `commands/import_export.rs`:
     - `"insomnia"` -> `export_to_insomnia_v5` (YAML).
     - `"insomnia-v4"` -> `export_to_insomnia_v4` (JSON).

2. **V5 YAML Serializer**:
   - Build recursive tree node structure:
     - Folders emit `name`, `meta: { id, created, modified }`, `children`.
     - Requests emit `url`, `name`, `meta`, `method`, `headers`, `parameters`, `body`, `authentication`, `scripts`, `settings`.
   - Convert variable references in URL/headers/body:
     - `add_insomnia_env_prefix(input: &str) -> String`:
       - `{{ var }}` -> `{{ _.var }}` (if `var` does not start with `$`).
   - Read `.apiark/environments/*.yaml` files:
     - If `Base.yaml` exists -> use as root `data`.
     - Other files -> write to `subEnvironments`.
     - If no `Base.yaml` -> write first environment as root or empty base with sub-environments.
   - Serialize with `serde_yaml::to_string`.

3. **CLI Exporter**:
   - Support `--format insomnia` (v5 YAML) and `--format insomnia-v4` (v4 JSON).

---

### 3.5 UI & Type Definitions

1. **`packages/types/src/index.ts`**:
   ```typescript
   export type ExportFormat = "postman" | "openapi" | "apiark" | "bruno" | "insomnia" | "insomnia-v4";
   ```
2. **`apps/desktop/src/lib/export-collection.ts`**:
   - `"insomnia"`: `{ ext: "insomnia.yaml", filterName: "YAML", filterExt: ["yaml", "yml"] }`
   - `"insomnia-v4"`: `{ ext: "insomnia.json", filterName: "JSON", filterExt: ["json"] }`
3. **`apps/desktop/src/components/import/import-dialog.tsx`**:
   - Label: `"Insomnia (v4 / v5)"`.
4. **Locales (`apps/desktop/src/locales/*.json`)**:
   - Update `"insomnia": "Insomnia (v4 / v5)"`.

---

## 4. Verification and Acceptance Criteria

1. **Live User Sample**:
   - `18576-1791954559473366900-Molect-Podi-v5-no-env.yaml` passes detection and imports:
     - Exactly 179 requests.
     - Exactly 26 folders.
     - 4 environments (`Base`, `Local`, `Prod`, `Dev`).
     - Sub-environment variables properly inherit `base_url` and `base_req`.
     - URLs and bodies have `{{ _.base_url }}` normalized to `{{ base_url }}`.
2. **CLI Runner**:
   - `apiark import <file> -o <dir>` imports v5 YAML seamlessly without errors.
3. **Roundtrip Export**:
   - Export collection to Insomnia v5 YAML -> re-imported into ApiArk yields identical request count, URLs, headers, and bodies.
4. **Unit Tests**:
   - Automated tests added to `apps/desktop/src-tauri` and `apps/cli` verifying:
     - Detection of v4 JSON, v4 YAML, v5 YAML, v5 JSON.
     - Variable stripping and re-scoping.
     - Environment merging with overrides.
     - Script extraction (`scripts.preRequest`, `scripts.afterResponse`).
