Contribution Number: 1
Student: Areeb Ehsan
Issue: graphql-hive/console#6954
Status: Phase IV — Pull Request Submitted (graphql-hive/console#8166)
Pull Request: https://github.com/graphql-hive/console/pull/8166

## Why I Chose This Issue

I chose GraphQL Hive Console issue #6954 because it is a meaningful developer-experience improvement in a real open source GraphQL platform. My understanding is that when the main schema does not change but a schema contract does change, the GitHub CI message currently displays "No changes," which can mislead users into thinking nothing relevant changed. A better message should either list the schema contract changes or clearly indicate that schema contracts changed.

This issue interests me because it is more substantial than a simple documentation typo, but still appears scoped enough for a first open source contribution. It matches my JavaScript/TypeScript background and gives me an opportunity to learn how a production codebase generates CI feedback, handles schema checks, and communicates changes to developers. I chose it because the issue is open, labeled as a good first issue, unassigned, and had no linked pull request or branch at the time of selection.

## Understanding the Issue

### Problem Description

When a composite/federation project's schema check finds no changes to the core composed schema but a schema contract did change, the GitHub CI check reports "No changes" — which is misleading, since the contract did change.

### Expected Behavior

The GitHub CI check should reflect that a contract changed (e.g., list the contract-level changes or clearly indicate that schema contracts changed).

### Current Behavior

The check title/summary says "No changes" / "No changes detected", even though `state.contracts[].schemaChanges.all` is populated.

### Affected Components

- `packages/services/api/src/modules/schema/providers/schema-publisher.ts` — `updateGithubCheckRunForSchemaCheck` call site (~line 954) and summary builder (~lines 3100-3104)
- `packages/services/api/src/modules/schema/providers/models/composite.ts` — `check()` (~lines 272-298), produces `SchemaCheckSuccess` with `state.schemaChanges` and `state.contracts[]`
- `packages/services/api/src/modules/schema/providers/models/shared.ts` — defines the per-contract `schemaChanges` shape (`{ breaking, safe, all }`)

## Reproduction Process

### Environment Setup

Large pnpm monorepo on Windows 11. Setup challenges and resolutions:

| Challenge | How I resolved it |
| --- | --- |
| `pnpm` not on PATH | Used Corepack (`corepack pnpm ...`), which honors the pinned `pnpm@10.33.2` in `package.json`. No separate global install needed. |
| Node version mismatch (real blocker): `package.json` `engines` requires `node >=24.14.1`, `.node-version` pins `24.14.1`, but system had v22.20.0. `.npmrc` sets `engine-strict=true`, so pnpm refuses to install on the wrong Node. | Installed **fnm** via `winget install Schniz.fnm`, then `fnm install 24.14.1` and `fnm use 24.14.1` to pin the exact required version without disturbing system Node. |
| Root env file | Created root `.env` with `ENVIRONMENT=local` as documented in `docs/DEVELOPMENT.md`. |
| `corepack pnpm install` exits 1 on final `cargo:fix` postinstall step (`bash ./scripts/fix-symbolic-link.sh` fails because PowerShell's `bash` hits a broken WSL) | Confirmed harmless — the script is a no-op outside Cygwin, and all 3,465 packages link successfully before this step. `node_modules` is fully populated. |

Install + baseline test commands (run from repo root, Node 24.14.1 active):

```bash
corepack pnpm install
corepack pnpm vitest run packages/services/api/src/modules/schema
```

### Steps to Reproduce

Scenario: a federation/composite project that has at least one schema contract. A developer opens a PR that changes *only* the contract's schema (e.g., adds a field tagged so it lands in the contract), leaving the core composed schema unchanged. They run `hive schema:check --github`.

1. A composite schema check that touches only a contract returns a `SchemaCheckSuccess` from `packages/services/api/src/modules/schema/providers/models/composite.ts` (`check()`, lines ~272-298). The top-level `state.schemaChanges` is empty/null (core schema unchanged), but `state.contracts[].schemaChanges.all` **is populated** (the contract changed).
2. The check result flows to `SchemaPublisher` in `packages/services/api/src/modules/schema/providers/schema-publisher.ts`. In the `Success` branch, the call to `updateGithubCheckRunForSchemaCheck(...)` passes only the core changes:
```ts
   // schema-publisher.ts:954
   changes: checkResult.state?.schemaChanges?.all ?? null,
```
   Contract changes (`checkResult.state.contracts[].schemaChanges`) are never passed. (Line 946 already reads `state.contracts`, but only to count failed compositions — not changes.)
3. Inside `updateGithubCheckRunForSchemaCheck`, the summary is built from that single `changes` value:
```ts
   // schema-publisher.ts:3100-3104
   if (conclusion === SchemaCheckConclusion.Success) {
     if (!changes || changes.length === 0) {
       title = 'No changes';
       summary = 'No changes detected';
```
   Because `changes` is empty (core unchanged) and contract changes were dropped at step 2, this branch always produces "No changes" — the bug.

- **Observed result:** The GitHub check reports "No changes" even though the contract did change. This is deterministic — for any check where core schema is unchanged but a contract changed, the same branch is taken every time.

### Reproduction Evidence

- **Commit/branch showing reproduction setup:** https://github.com/AreebEhsan/graphql-hive-console/tree/fix-issue-6954
- **Logs:** Baseline `@hive/api` schema-module unit tests on the unmodified branch:
- corepack pnpm vitest run packages/services/api/src/modules/schema

→ 2 test files, 16 tests passed (3.36s, Node 24.14.1)


- **My findings:** There is no UI/CLI click-through reproduction available for this bug. The GitHub check summary only renders when `hive schema:check --github` runs against a project with a live GitHub App installation. Standing that up locally requires a public tunnel (loophole/ngrok), creating a GitHub App, and installing it on a test repo (see `docs/DEVELOPMENT.md` → "Setting up GitHub App for developing"). The project's own automated tests don't stand this up either. For a backend-logic bug like this, the credible reproduction is a precise code trace of the faulty branch (above) plus the passing baseline test run — the standard way to reproduce logic that can't be triggered through the UI.

## Solution Approach

### Analysis

Root cause: the GitHub check summary is built from core schema changes only. Per-contract changes already exist on the check result (`checkResult.state.contracts[].schemaChanges`) but are dropped at the `SchemaPublisher` call site and never reach the summary builder. The "No changes" condition only checks the (empty) core `changes` array.

### Proposed Solution

Thread per-contract change data into `updateGithubCheckRunForSchemaCheck` and fix the "No changes" condition so it accounts for contract-level changes too, mirroring the existing pattern already used in this file for `failedContractCompositionCount`.

### Implementation Plan (UMPIRE)

**Understand:** When `schema:check --github` runs on a composite/federation project and only a contract's schema changed (core composed schema unchanged), the GitHub check reports "No changes". It should instead reflect that a contract changed. Root cause: the summary builder only sees core schema changes; per-contract changes are dropped at the call site.

**Match:** The same file already plumbs contract data into this exact function for a different purpose: `failedContractCompositionCount` is computed from `checkResult.state.contracts` (schema-publisher.ts:945-946) and passed as a parameter into `updateGithubCheckRunForSchemaCheck`, where it's rendered into the failure summary (schema-publisher.ts:3117-3119, 3134-3136). I will follow this exact pattern — derive contract-change info from `state.contracts` at the call site and thread it in as a new parameter. The per-contract `schemaChanges` shape (`{ breaking, safe, all }`) is defined in `models/shared.ts` (`SchemaCheckSuccess.state.contracts[]`).

**Plan:**
1. Add a parameter to `updateGithubCheckRunForSchemaCheck` carrying per-contract changes, e.g. `contractChanges: Array<{ contractName: string; changes: Array<SchemaChangeType> }> | null` (grouped by contract so the summary can attribute changes to the contract that changed).
2. At the `Success` call site (schema-publisher.ts:949-962), derive that value from `checkResult.state.contracts` (filter to contracts whose `schemaChanges.all` is non-empty) and pass it in.
3. In the `Success` branch of the summary builder (schema-publisher.ts:3100), fix the condition so "No changes" is only shown when **both** core changes **and** contract changes are empty. Otherwise build a summary that includes a per-contract changes section (reusing the existing `changesToMarkdown` helper).
4. (Optional, for testability) Extract the title/summary-building logic into a small pure helper so it can be unit-tested directly without the full `SchemaPublisher` dependency-injection graph.

**Implement:** Done (Phase III). Branch `fix-issue-6954`, four atomic commits:
1. `refactor(schema-publisher): add contractChanges param to github check output`
2. `refactor(schema-publisher): derive contract changes at schema check call site`
3. `fix(schema-publisher): include contract changes in github check summary`
4. `chore: add changeset for contract changes in github check summary`

Plan step 4 (the optional pure-helper extraction) was implemented, since no DI/mocking harness exists for the providers — extracting `buildSchemaCheckSuccessGithubOutput` (with the markdown renderer injected) was the only way to unit-test in the repo's house style. A `'hive': patch` changeset was added (the convention used for server-side platform fixes; `@hive/api` itself is in the changeset `ignore` list).

**Review:**
- Self-reviewed against `docs/CONTRIBUTING.md` (which points to the-guild's Hive contributing guide) and the repo's commit/PR conventions before opening the PR.
- Added a `'hive': patch` Changeset describing the user-facing fix, matching the convention used by recent server-side fixes in the repo.
- Kept the change minimal and matched to the surrounding style (the existing `failedContractCompositionCount` plumbing was the template). No drive-by refactors; the other branches' `changesToMarkdown` calls are untouched.

**Evaluate:** Done. Behavior preserved byte-for-byte for every project where no contract changed (verified by hand and by a dedicated regression test). Verified with `corepack pnpm vitest run packages/services/api` (97/97), `tsc --noEmit -p packages/services/api/tsconfig.json` (0 errors), and ESLint on the changed files (0 problems). The "truly no changes" case still reports "No changes".

## Testing Strategy

### Unit Tests

New tests live in `packages/services/api/src/modules/schema/providers/schema-publisher.spec.ts`, targeting the extracted pure helper `buildSchemaCheckSuccessGithubOutput`:

- [x] Core changes empty + contract changes present → title is **not** "No changes" and summary mentions the contract name
- [x] Core changes empty + contract changes empty → "No changes" still correctly reported (regression guard for existing behavior)
- [x] Null core changes + null contract changes → "No changes"
- [x] Contracts present but with empty change lists → "No changes"
- [x] Core-only changes (no contracts) → byte-for-byte identical to previous behavior
- [x] Combined core + per-contract changes → both listed in the summary
- [x] Existing `@hive/api` schema-module suite continues to pass

Results: schema-module scope now **3 files / 22 tests passing** (was 2 / 16); full `@hive/api` unit suite **12 files / 97 tests passing**; `tsc --noEmit` clean; ESLint clean.

### Integration Tests

- [x] Assessed: not applicable. The Docker-based integration suite requires a live GitHub App and does not exercise this pure logic. The fix is fully covered at the unit layer.

### Manual Testing

Not performed — a click-through reproduction requires a live GitHub App installation (out of scope, see Reproduction Evidence). Verification instead relies on the unit tests above, the full `@hive/api` suite, the type-check, and the lint pass.

## Implementation Notes

### Week 2 Progress (Phase II)

Set up the local development environment on Windows: resolved a Node version mismatch (system had v22.20.0, repo requires `>=24.14.1` with `engine-strict=true`) by installing fnm and pinning Node 24.14.1, used Corepack to run the pinned pnpm version, and completed `pnpm install` (3,465 packages linked; the one postinstall failure on `cargo:fix` was confirmed harmless — a no-op outside Cygwin). Ran the baseline `@hive/api` schema-module test suite (16/16 passing). Produced a deterministic code-trace reproduction of the bug (anchored to `schema-publisher.ts:954` and `:3100-3104`) and a full UMPIRE implementation plan modeled on the existing `failedContractCompositionCount` pattern in the same file.

### Week 3 Progress (Phase III)

Implemented the fix in four small, independently-verified commits (each ran the schema test scope + type-check before the next). Added the `contractChanges` parameter to `updateGithubCheckRunForSchemaCheck` mirroring the existing `failedContractCompositionCount` plumbing, derived it from `checkResult.state.contracts` at the success call site, and extracted the success-branch title/summary logic into a pure, testable helper `buildSchemaCheckSuccessGithubOutput` (markdown renderer injected so it can be tested without the full `SchemaPublisher` DI graph). "No changes" now only shows when neither the core schema nor any contract changed; otherwise the summary lists core changes followed by a per-contract section. Wrote 6 unit tests (required cases + edge cases). Final state: 97/97 api unit tests, clean `tsc`, clean ESLint, `'hive': patch` changeset. Scope kept to one source file + one new test file + the changeset; no drive-by refactors.

### Week 4 Progress (Phase IV)

Opened pull request graphql-hive/console#8166 (base `main` ← `AreebEhsan:fix-issue-6954`, +153 / −9 across 3 files), with a description following the repo's PR template and a `Fixes #6954` reference. The automated Gemini Code Assist review returned no feedback ("I have no feedback to provide"). Awaiting CI results and human maintainer review; will iterate on the same branch as feedback arrives (pushes auto-update the PR).
