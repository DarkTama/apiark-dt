# Agent Instructions: Fork Maintenance & Branching Rules

## 1. Remote Architecture
- `origin`: Fork repository (`DarkTama/apiark-dt`)
- `upstream`: Original repository (`berbicanes/apiark`)

## 2. Branch Roles
- `upstream-sync`:
  - 1:1 pristine mirror of `upstream/main`.
  - NEVER commit custom code here. Fast-forward only.
- `main`:
  - Primary working branch of fork containing custom features, UI changes, patches, and internal docs (`GLOSSARY.md`, `SPEC-*.md`, `docs/adr/`).
  - Receives updates by merging `upstream-sync`.
- `feat/*`:
  - Short-lived feature branches for upstream PRs.
  - MUST branch directly off `upstream/main` (or `upstream-sync`), NOT from `main`.
  - Keeps internal docs and unrelated fork changes out of upstream PRs.
  - One feature per branch. One or few clean atomic commits.

## 3. Upstream Sync Protocol
When pulling new updates from upstream:
```bash
# 1. Update clean mirror
git checkout upstream-sync
git pull upstream main
git push origin upstream-sync

# 2. Integrate into fork main
git checkout main
git merge upstream-sync
# Resolve conflicts if any, test, then:
git push origin main
```

## 4. Upstream Pull Request Protocol
When creating PR targeting upstream:
1. Ensure branch starts from clean `upstream-sync`:
   ```bash
   git checkout -b feat/<name> upstream-sync
   ```
2. Apply changes and verify tests pass:
   ```bash
   cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml --lib
   cargo test --manifest-path apps/cli/Cargo.toml
   pnpm test
   ```
3. Push branch to `origin`:
   ```bash
   git push -u origin feat/<name>
   ```
4. Open PR targeting `upstream:main`:
   ```bash
   gh pr create --repo berbicanes/apiark --base main --head DarkTama:feat/<name>
   ```

## 5. Release Workflow
- Releases triggered by git tag matching pattern `v*` or manual workflow dispatch in GitHub Actions.
- Tag from `main`:
  ```bash
  git checkout main
  git tag v<semver>-dt.<patch>
  git push origin v<semver>-dt.<patch>
  ```
