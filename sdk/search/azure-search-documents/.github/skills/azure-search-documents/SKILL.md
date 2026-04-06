---
name: azure-search-documents
description: "**WORKFLOW SKILL** — Orchestrate the full release cycle for azure-search-documents SDK including TypeSpec generation, testing, versioning, and pre-release validation. WHEN: \"search SDK release\", \"regenerate search SDK\", \"update search API version\", \"search pre-release validation\", \"fix search test failures\", \"search changelog\". REQUIRES: azure-sdk-mcp tools. FOR SINGLE OPERATIONS: Use azsdk MCP tools directly."
---

# azure-search-documents — Package Skill

## When to Use This Skill

Activate when user wants to:
- Prepare a new GA or preview release
- Regenerate SDK from TypeSpec (new API version or current spec)
- Run pre-release validation (tests, linting, checks)

## Prerequisites

- Read [references/architecture.md](references/architecture.md) for source layout, clients, branching, and CHANGELOG conventions
- Read [references/testing.md](references/testing.md) for running tests and writing new tests
- Azure SDK MCP server must be running (provides `azsdk_*` tools)

## Environment Setup (MANDATORY — run before ANY command)

```powershell
. .\.github\skills\azure-search-documents\scripts\activate-venv.ps1
```

This activates the venv, puts `azpysdk` on PATH, and **exits the shell if activation fails**. Use a **single persistent async shell** for all commands so venv stays active. If the shell dies, re-run this script in the new shell before doing anything else.

## Steps

### Phase 0 — Determine Scope

Ask the user:
1. New API version or regeneration of current spec?
2. GA release or beta/preview release?
3. Target version number and release date?

If new API version: get the spec **commit SHA**, **API version string** (e.g., `2026-04-01`), and the **spec PR link** (e.g., a PR in `Azure/azure-rest-api-specs`).

> **STOP** if the user cannot provide the commit SHA. Do not guess or use HEAD.

### Phase 1 — Generate SDK

1. If commit SHA is changing, update `commit` in `tsp-location.yaml`.
2. Use `azsdk_package_generate_code` to generate SDK from TypeSpec.

### Phase 2 — Sync Handwritten Code with Generated Code

After regeneration, handwritten code must be synced with whatever changed in the generated layer. Use `git diff` as the source of truth — do not guess. The spec PR's CHANGELOG can help understand what changed, but always verify against the actual generated code (see source of truth priority in Phase 5).

**1. Get the full diff of generated code:**

```bash
git diff origin/main -- azure/search/documents/ ':!*/_patch.py'
```

Keep this diff handy — it tells you exactly what was added, removed, or renamed.

**2. Loop through all 15 `_patch.py` files** (see full map in [references/architecture.md](references/architecture.md)). For each one, check against the diff:

- **Removed** APIs → remove all references. Grep the name across ALL `_patch.py` files to catch the entire call chain (wrapper classes delegate to inner classes).
- **Added** APIs → add convenience wrappers, re-exports to `__all__`, or model overrides as needed.
- **Renamed** APIs → adopt the new name everywhere.
- **Sync + async** — every sync `_patch.py` has an async mirror. Apply identical changes to both.
- **Search both** Python snake_case (`debug_info`) and JSON camelCase (`debugInfo`), case-insensitive.

**3. Update all version strings in `_patch.py` files** — `ApiVersion` enum, `DEFAULT_VERSION`, and `api_version` docstrings must reflect the new API version. Check both sync and async `_patch.py` files.

**4. Update `apiview-properties.json`** if the public API surface changed.

### Phase 3 — Update and Run Tests

**1. Loop through all test files** with the Phase 2 generated-code diff handy. For each one:

- **Removed** APIs → remove or rewrite tests that call them.
- **Added** APIs → add new tests for new features or operations.
- **Renamed** APIs → update test references to use the new names.
- Check both sync and async test files.

**2.** Run unit tests, then live tests. See [references/testing.md](references/testing.md) for commands and recording workflow.

**Gate:** All unit and live tests pass.

### Phase 4 — Build, Lint, and Checks

Use MCP tools to validate:

1. `azsdk_package_build_code` — build and detect compilation errors.
2. `azsdk_package_run_check` with `Linting` — run pylint and mypy. Fix any errors in `_patch.py` files.
3. `azsdk_package_run_check` with `Changelog`, `Cspell`, `Snippets` — run remaining checks.

> **Exception:** For pylint issues in **generated code** (not `_patch.py`), you MAY add inline `# pylint: disable=<rule>` comments directly in the generated `.py` files. This is the **only** case where editing generated code is allowed.

**Gate:** Build, lint, and all checks pass.

### Phase 5 — Update Changelog

Follow the CHANGELOG conventions in [references/architecture.md](references/architecture.md). Use `azsdk_package_update_changelog_content` to draft entries, then review and adjust.

The spec PR's CHANGELOG is a useful reference, but apply this **source of truth priority**:
1. **Auto-generated code** — always first. If it exists in generated code, treat it as present.
2. **TypeSpec config in the spec PR** — second. Defines the intended API shape.
3. **CHANGELOG in the spec PR** — third. Useful for context, but may lag behind.

#### Mapping spec PR to SDK CHANGELOG (GA releases only)

The spec PR has 4 sections by **comparison basis**. Our SDK CHANGELOG has 3 sections by **change type**:

- **Features Added** ← spec PR's "GA-to-GA Non-Breaking Changes"
- **Breaking Changes (before disclaimer)** ← spec PR's "GA-to-GA Breaking Changes"
- **Breaking Changes (after disclaimer)** ← spec PR's "Preview-to-GA Breaking Changes"

"Preview-to-GA Non-Breaking Changes" from the spec PR are intentionally not included.

> For preview releases, the spec PR may use a different structure — adapt accordingly.

#### 3-way verification (actual code → our CHANGELOG → spec PR CHANGELOG)

After drafting the CHANGELOG, do a systematic cross-check following the source of truth priority above:

1. **Code → our CHANGELOG**: For every item in our CHANGELOG, verify it matches actual code (e.g., removed properties are truly absent, new models actually exist). Run Python imports or `getattr` checks.
2. **Spec PR → our CHANGELOG**: For every item in the spec PR's CHANGELOG, confirm it's covered in ours (or intentionally excluded per the mapping rules above).
3. **Our CHANGELOG → code**: For any item we added that isn't in the spec PR (Python-specific changes), verify it's real.

#### Sorting

All lists within each CHANGELOG section should be sorted alphabetically by fully qualified name.

### Phase 6 — Update Samples

Update existing samples and add new samples for any new features or changed APIs.

### Phase 7 — Update Version and Metadata

Use `azsdk_package_update_version` and `azsdk_package_update_metadata` to bump the version and update package metadata.

## Reference Files

| File | Contents |
|------|----------|
| [references/architecture.md](references/architecture.md) | Source layout, clients, branching, CHANGELOG conventions |
| [references/testing.md](references/testing.md) | Running tests, writing new tests, test recording |
