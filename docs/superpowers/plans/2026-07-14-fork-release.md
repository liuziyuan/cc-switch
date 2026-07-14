# Fork Release Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build macOS universal and Windows x64 releases from any branch in `liuziyuan/cc-switch`, with an isolated Tauri updater channel.

**Architecture:** Keep business branches identical to the upstream PR while storing fork-only release infrastructure on the fork default branch. A manually dispatched workflow checks out `source_ref`, derives a numeric prerelease app version, injects the fork updater public key and endpoint at build time, publishes signed updater artifacts, and assembles `latest.json`.

**Tech Stack:** GitHub Actions, Tauri 2, pnpm, Rust, Bash, PowerShell, GitHub CLI

## Global Constraints

- GitHub release tags are `v3.17.0-copilot-codex`, then `v3.17.0-copilot-codex.2`, `.3`, and so on.
- Embedded versions are numeric prereleases: `3.17.0-1`, `3.17.0-2`, and so on, so WiX can convert them to numeric MSI versions.
- Releases are not prereleases because GitHub's `/releases/latest/` endpoint excludes prereleases.
- Only macOS universal and Windows x64 are built.
- The app name, identifier, data directory, and deep-link scheme remain unchanged.
- The default `src-tauri/tauri.conf.json` remains pointed at the official updater.
- Fork private signing keys must never be committed.

---

### Task 1: Fork Signing Identity

**Files:**
- Create outside repository: `~/.cc-switch/fork-updater.key`
- Create outside repository: `~/.cc-switch/fork-updater.key.pub`
- Create outside repository: `~/.cc-switch/fork-updater.key.password`
- Configure: GitHub Actions environment `fork-release`, restricted to branch `main`
- Configure in that environment: secret `TAURI_SIGNING_PRIVATE_KEY`
- Configure in that environment: secret `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`
- Configure: GitHub Actions variable `FORK_UPDATER_PUBKEY`

**Interfaces:**
- Produces: an encrypted private key and password available to the trusted signing job, plus a matching public key available as `${{ vars.FORK_UPDATER_PUBKEY }}`.

- [ ] **Step 1: Generate a password-protected updater key pair**

Generate a random hexadecimal password outside the repository, then run `pnpm tauri signer generate -w ~/.cc-switch/fork-updater.key -p "$PASSWORD" --ci`.

Expected: private and public key files are created outside the repository.

- [ ] **Step 2: Configure the fork repository**

Run: `gh secret set TAURI_SIGNING_PRIVATE_KEY --repo liuziyuan/cc-switch --env fork-release < ~/.cc-switch/fork-updater.key`

Run: `gh secret set TAURI_SIGNING_PRIVATE_KEY_PASSWORD --repo liuziyuan/cc-switch --env fork-release < ~/.cc-switch/fork-updater.key.password`

Run: `gh variable set FORK_UPDATER_PUBKEY --repo liuziyuan/cc-switch --body "$(cat ~/.cc-switch/fork-updater.key.pub)"`

Expected: `gh secret list` and `gh variable list` show both names without exposing the private value.

### Task 2: Fork Release Workflow

**Files:**
- Create: `.github/workflows/fork-release.yml`
- Modify: `.github/workflows/release.yml:15-18`

**Interfaces:**
- Consumes: `source_ref`, `base_version`, `sequence`, `TAURI_SIGNING_PRIVATE_KEY`, and `FORK_UPDATER_PUBKEY`.
- Produces: tag `v<base>-copilot-codex[.<sequence>]`, macOS and Windows assets, and `latest.json` with embedded version `<base>-<sequence>`.

- [ ] **Step 1: Add repository guard to official workflow**

Add `if: github.repository == 'farion1231/cc-switch'` to the official release job.

- [ ] **Step 2: Add dispatch validation and metadata job**

Validate `base_version` against `^[0-9]+\.[0-9]+\.[0-9]+$`, `sequence` against `^[1-9][0-9]*$`, require the updater key variable and secret, resolve `source_ref`, and derive the tag/build version.

- [ ] **Step 3: Add unprivileged platform builds and isolated signing**

Build with read-only permissions, no persisted Git credentials, and no signing secrets. A separate trusted job downloads the resulting updater artifacts and signs them with the fork key.

- [ ] **Step 4: Add release publishing and latest metadata**

Upload installer and updater files as workflow artifacts, generate `latest.json`, upload every asset to a draft GitHub Release, then publish that draft as the non-prerelease latest release.

- [ ] **Step 5: Commit workflow infrastructure**

Run: `git add .github/workflows/release.yml .github/workflows/fork-release.yml docs/superpowers/plans/2026-07-14-fork-release.md && git commit -m "ci(release): add fork distribution workflow"`

Expected: one commit containing fork-only release infrastructure.

### Task 3: Static Verification and Delivery

**Files:**
- Verify: `.github/workflows/fork-release.yml`
- Verify: `.github/workflows/release.yml`

**Interfaces:**
- Consumes: workflow created by Task 2.
- Produces: validated workflow on fork `main` and a clean upstream feature branch.

- [ ] **Step 1: Parse YAML and validate workflow rules**

Run Ruby YAML parsing with aliases enabled, check for repository guards, exact updater URL, numeric embedded version construction, and non-prerelease publishing.

Expected: all assertions pass.

- [ ] **Step 2: Run project checks**

Run: `pnpm typecheck`

Expected: PASS. Record the known baseline `tests/integration/App.test.tsx` failures rather than attributing them to workflow-only changes.

- [ ] **Step 3: Push infrastructure and merge into fork main**

Push `chore/fork-release-infrastructure`, merge it into `liuziyuan/cc-switch:main` through a fork PR, and verify the workflow is visible from the default branch.

- [ ] **Step 4: Clean and push the business branch**

Remove the design-only commit from `feat/codex-github-copilot` without touching business commits or unrelated user changes, then push the clean branch to `fork`.

Expected: upstream comparison contains business changes only.

### Task 4: First Release Integration

**Files:**
- Runtime output: GitHub Release `v3.17.0-copilot-codex`

**Interfaces:**
- Consumes: default-branch workflow and `source_ref=feat/codex-github-copilot`.
- Produces: installable macOS/Windows packages and fork `latest.json` version `3.17.0-1`.

- [ ] **Step 1: Dispatch first release**

Run: `gh workflow run fork-release.yml --repo liuziyuan/cc-switch --ref main -f source_ref=feat/codex-github-copilot -f base_version=3.17.0 -f sequence=1`

- [ ] **Step 2: Wait for the workflow**

Use `gh run watch --exit-status`.

Expected: metadata, macOS, Windows, publish, and latest metadata jobs pass.

- [ ] **Step 3: Validate release metadata**

Confirm tag `v3.17.0-copilot-codex`, required assets, `latest.json.version == "3.17.0-1"`, fork URLs, and complete signatures for `darwin-aarch64`, `darwin-x86_64`, and `windows-x86_64`.

- [ ] **Step 4: Commit no runtime artifacts**

Expected: downloaded validation files remain outside the repository and repository status stays clean.
