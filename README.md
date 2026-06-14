Contribution Number: 1
Student: Areeb Ehsan
Issue: graphql-hive/console#6954
Status: Phase II Complete

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

**Implement:** Branch: https://github.com/AreebEhsan/graphql-hive-console/tree/fix-issue-6954 (Phase III)

**Review:**
- Self-review against `docs/CONTRIBUTING.md` (which points to the-guild's Hive contributing guide) and the repo's commit/PR conventions before opening the PR.
- The repo uses Changesets — I'll add a changeset describing the user-facing fix if the contribution guidelines require one for this package.
- Keep the change minimal and matched to the surrounding style (the existing `failedContractCompositionCount` plumbing is my template).

**Evaluate:**
- Add a unit test asserting that when core changes are empty but contract changes are present, the produced GitHub check title is not "No changes" and the summary mentions the contract. (If the pure helper from Plan step 4 is extracted, this test targets that helper directly.)
- Run the `@hive/api` test suite to confirm no regressions: `corepack pnpm test --filter @hive/api`.
- Confirm the existing "truly no changes" case (no core changes AND no contract changes) still reports "No changes".

## Testing Strategy

### Unit Tests

- [ ] Core changes empty + contract changes present → title is not "No changes" and summary mentions the contract
- [ ] Core changes empty + contract changes empty → "No changes" still correctly reported (regression guard for existing behavior)
- [ ] Existing `@hive/api` schema-module suite continues to pass (baseline: `corepack pnpm vitest run packages/services/api/src/modules/schema` → 2 files, 16 tests passed)

### Integration Tests

- [ ] Not yet planned — to be assessed during Phase III implementation

### Manual Testing

Not performed. A full click-through reproduction requires a live GitHub App installation (public tunnel via loophole/ngrok, GitHub App creation, and installation on a test repo per `docs/DEVELOPMENT.md`), which is out of scope for this contribution. Verification instead relies on the code trace, the implementation plan's unit tests, and the existing test suite.

## Implementation Notes

### Week 2 Progress (Phase II)

Set up the local development environment on Windows: resolved a Node version mismatch (system had v22.20.0, repo requires `>=24.14.1` with `engine-strict=true`) by installing fnm and pinning Node 24.14.1, used Corepack to run the pinned pnpm version, and completed `pnpm install` (3,465 packages linked; the one postinstall failure on `cargo:fix` was confirmed harmless — a no-op outside Cygwin). Ran the baseline `@hive/api` schema-module test suite (16/16 passing). Produced a deterministic code-trace reproduction of the bug (anchored to `schema-publisher.ts:954` and `:3100-3104`) and a full UMPIRE implementation plan modeled on the existing `failedContractCompositionCount` pattern in the same file.
