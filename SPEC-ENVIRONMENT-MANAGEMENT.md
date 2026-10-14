# Technical Specification: Environment Management & Multi-Collection Lifecycle Fixes

- **Document:** SPEC-ENVIRONMENT-MANAGEMENT
- **Status:** Approved for Implementation
- **Target Components:** 
  - `apps/desktop/src-tauri/src/storage/environment.rs`
  - `apps/desktop/src-tauri/src/commands/environment.rs`
  - `apps/desktop/src-tauri/src/exporter/insomnia.rs`
  - `apps/desktop/src/stores/environment-store.ts`
  - `apps/desktop/src/stores/collection-store.ts`
  - `apps/desktop/src/stores/tab-store.ts`
  - `apps/desktop/src/components/environment/environment-selector.tsx`
  - `apps/desktop/src/components/layout/side-panel.tsx`
  - `apps/desktop/src/components/request/url-bar.tsx`

---

## 1. Problem Statement

Testing uncovered 8 interdependent issues in environment persistence, multi-collection lifecycle, and UI layout:

1. **Environment Duplication on Edit/Rename/Scope Change**:
   - `save_environment()` writes to `.apiark/<subdir>/<filename>.yaml` using the current name.
   - When renaming (`Dev` -> `Development`) or toggling scope (`Shared` -> `Personal`), the old file on disk is never deleted.
   - Both files are read by `load_environments()`, creating duplicate UI entries.

2. **Casing & Filename Inconsistency**:
   - Importers write original casing (e.g. `Dev.yaml`).
   - UI editor saves slugified lowercase (e.g. `dev.yaml`).
   - On Linux (case-sensitive) or Windows renames, both files coexist.

3. **In-Memory Environment Leak on Collection Deletion / Close**:
   - `closeCollection()` and `deleteCollection()` remove the collection tree and close open tabs, but never reset `useEnvironmentStore`.
   - `environments`, `activeEnvironmentName`, and `runtimeOverrides` from deleted collections remain indefinitely in React state.

4. **Hardcoded First Collection Binding**:
   - `side-panel.tsx` and `environment-selector.tsx` resolve collection path via `collections[0]` or `collections.find(c => c.type === 'collection')`.
   - When multiple collections are open, the sidebar always displays and edits environments of the first collection, ignoring the active tab's collection.

5. **Sticky Environment Selector Across Collections**:
   - `loadEnvironments()` sets: `activeEnvironmentName: get().activeEnvironmentName ?? ...`.
   - Switching from Collection A to Collection B retains Collection A's selection (e.g., `Dev`), causing invalid cross-collection variable lookups.

6. **Phantom Overwrites**:
   - Bugs #3, #4, and #5 combine: deleting Collection A, importing Collection B, and hitting "Save" in the side panel writes stale Collection A environments into Collection B's `.apiark/environments/`.

7. **Export Privacy Leak**:
   - Collection exporters call `load_environments()`, which indiscriminately loads both `environments/` (shared) and `environments.local/` (personal gitignored secrets), leaking personal credentials into export YAML/JSON files.

8. **Missing Environment Switcher in URL Bar**:
   - The environment dropdown is only available in the left sidebar. Collapsing the sidebar blinds the user to the active environment context during request editing and execution.

---

## 2. Technical Architecture & Component Changes

### 2.1 Backend: Environment Persistence & Cleanup (`apps/desktop/src-tauri`)

#### 1. `storage/environment.rs`
- Update `save_environment` signature:
  ```rust
  pub fn save_environment(
      collection_path: &Path,
      env: &EnvironmentFile,
      original_name: Option<&str>,
      original_scope: Option<&EnvironmentScope>,
  ) -> Result<(), String>
  ```
- **Old File Deletion**:
  - If `original_name` or `original_scope` are provided:
    - Determine old subdirectory (`environments` or `environments.local`).
    - Resolve old filename: scan old subdirectory for files matching `original_name` case-insensitively (handling `.yaml` and `.yml`).
    - Resolve new file destination.
    - If old path exists and differs from new path (distinct path on disk, not case-collision on case-insensitive filesystems), delete the old file.
- **Casing Normalization**:
  - Ensure uniform file naming: slugify to lowercase (`{name.to_lowercase().replace(' ', "-")}.yaml`).

#### 2. `commands/environment.rs`
- Update Tauri IPC command:
  ```rust
  #[tauri::command]
  pub async fn save_environment(
      collection_path: String,
      env: EnvironmentFile,
      original_name: Option<String>,
      original_scope: Option<EnvironmentScope>,
  ) -> Result<(), String>
  ```

#### 3. `exporter/insomnia.rs`
- In `export_environments_v5`:
  - Filter `load_environments(collection_path)` to **only** include `EnvironmentScope::Shared`.
  - Exclude personal local environments (`environments.local`) to prevent leaking private secrets.

---

### 2.2 Frontend Stores: Lifecycle Synchronization (`apps/desktop/src/stores`)

#### 1. `environment-store.ts`
- Add `clear()` action:
  ```ts
  clear: () => set({
    environments: [],
    activeEnvironmentName: null,
    activeCollectionPath: null,
    runtimeOverrides: {},
  })
  ```
- Update `loadEnvironments(collectionPath)`:
  - If `collectionPath !== get().activeCollectionPath`:
    - Check if new `envs` contain an environment matching `currentActiveName`.
    - If matching name exists in new collection, preserve it; otherwise reset to `envs[0]?.name ?? null`.
    - Clear `runtimeOverrides`.

#### 2. `tab-store.ts`
- In `setActiveTab(id)`:
  - Lookup active tab.
  - If tab has `collectionPath` and it differs from `useEnvironmentStore.getState().activeCollectionPath`:
    - Dispatch `useEnvironmentStore.getState().loadEnvironments(tab.collectionPath)`.
- When all tabs close, if no collections are open, call `useEnvironmentStore.getState().clear()`.

#### 3. `collection-store.ts`
- In `closeCollection(path)` and `deleteCollection(path, name)`:
  - If `useEnvironmentStore.getState().activeCollectionPath === path`:
    - Find remaining open collection (e.g. from active tab, or first remaining in `collections`).
    - If found: `loadEnvironments(remaining.path)`.
    - If no collections remain: `useEnvironmentStore.getState().clear()`.

---

### 2.3 UI Components: Active Collection Binding & URL Bar Switcher

#### 1. `apps/desktop/src/lib/tauri-api.ts`
- Update `saveEnvironment`:
  ```ts
  export async function saveEnvironment(
    collectionPath: string,
    env: EnvironmentData,
    originalName?: string,
    originalScope?: "shared" | "personal",
  ): Promise<void>
  ```

#### 2. `apps/desktop/src/components/layout/side-panel.tsx`
- Bind collection path dynamically:
  ```ts
  const activeTab = useTabStore((s) => s.tabs.find((t) => t.id === s.activeTabId));
  const collectionPath = activeTab?.collectionPath ?? collections[0]?.path ?? null;
  ```
- Pass `env.name` and `env.scope` as `originalName` and `originalScope` during save to remove old file on rename/scope-toggle.

#### 3. `apps/desktop/src/components/request/url-bar.tsx`
- Mount compact environment switcher left of the Send button:
  - Shows current environment name and color badge or dot.
  - Clicking opens select dropdown listing environments of the active collection.
  - Changing selection immediately updates `activeEnvironmentName` and re-evaluates URL segment overlay.

#### 4. `apps/desktop/src/components/environment/environment-selector.tsx`
- Accept optional `compact?: boolean` prop for URL bar rendering vs sidebar rendering.
- Bind to active collection path rather than static `collections[0]`.

---

## 3. Verification Criteria

1. **Rename Test**: Renaming environment `Dev` -> `Development` updates `.apiark/environments/development.yaml` and deletes `dev.yaml`/`Dev.yaml`. Zero duplicates.
2. **Scope Toggle Test**: Changing `Dev` from `Shared` to `Personal` moves file to `.apiark/environments.local/dev.yaml` and deletes from `environments/`.
3. **Deletion Test**: Deleting a collection clears memory. UI does not show ghost environments from deleted collection.
4. **Tab Switch Test**: Switching between Tab in Collection A and Tab in Collection B immediately switches active environments in URL bar and side panel.
5. **Export Privacy Test**: Exporting collection to Insomnia v5 excludes `.apiark/environments.local/` entries.
6. **URL Bar Switcher**: Environment can be changed directly from URL bar with sidebar collapsed.
