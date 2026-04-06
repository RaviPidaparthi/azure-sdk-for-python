# azure-search-documents Architecture

## Source Layout

Package: `sdk/search/azure-search-documents/`

```
tsp-location.yaml          # TypeSpec spec pointer (commit SHA, directory, repo)
pyproject.toml             # Package metadata, version, dependencies
CHANGELOG.md               # Release notes
assets.json                # Test recording tag
azure/search/documents/    # Source code (generated + handwritten)
tests/                     # All tests (sync, async, playback, live)
samples/                   # All samples (sync, async)
```

## Generated vs Handwritten

This SDK uses **TypeSpec** code generation. All files except `_patch.py` and `_version.py` are generated and overwritten on regeneration.

**Only edit `_patch.py` files** — they customize generated behavior (convenience methods, parameter normalization, response transformation, re-exports). Each sync `_patch.py` has an async mirror under `aio/`.

> **Docstrings:** When a `_patch.py` method wraps or overrides a generated method, copy the docstring from the generated code verbatim (parameter descriptions, types, return types). Do not paraphrase or abbreviate — consistency with the generated layer matters for API docs.

> **Exception:** Inline `# pylint: disable=` comments may be added to generated files for lint issues.

## `_patch.py` Map

4 sub-packages, 15 files total. All must be audited after every regeneration.

```
documents/                    # SearchClient
├── _patch.py                 # client-level patches
├── _operations/_patch.py     # search operations (sync)
├── models/_patch.py          # search models + re-exports
├── aio/_patch.py             # async client-level patches
├── aio/_operations/_patch.py # search operations (async)
├── indexes/                  # SearchIndexClient + SearchIndexerClient
│   ├── _patch.py
│   ├── _operations/_patch.py
│   ├── models/_patch.py
│   ├── aio/_patch.py
│   └── aio/_operations/_patch.py
└── knowledgebases/           # KnowledgeBaseRetrievalClient
    ├── _patch.py
    ├── _operations/_patch.py
    ├── models/_patch.py
    ├── aio/_patch.py
    └── aio/_operations/_patch.py
```

## Branching Strategy

Two parallel release tracks:

- **main** — latest preview. All new features and breaking changes. Preview releases are built from main.
- **GA branches** — built from the previous GA, not from main. Branch naming: `search/<api-version>-ga`.

GA releases create a PR targeting main for CI but **do not merge**. Close the PR after publishing.

## CHANGELOG Conventions

The preview-to-GA Breaking Changes section includes a disclaimer explaining **who is affected** — always people coming from the preview/beta track only.

### Preview releases

- **Features Added**: New relative to the **previous preview**.
- **Breaking Changes**: Relative to the previous preview. Start with:
  ```
  > These changes do not impact the API of stable versions such as <latest GA version>.
  > Only code written against a beta version such as <latest beta version> may be affected.
  ```

### GA releases

- **Features Added**: New relative to the **previous GA** (not the latest preview).
- **Breaking Changes** has two sections:
  1. **GA-to-GA breaking changes** (before disclaimer): Changes that break compatibility with the previous GA. Listed first, no disclaimer needed.
  2. **Preview-to-GA breaking changes** (after disclaimer): Preview-only items that did not graduate to GA. Start with:
     ```
     > These changes do not impact the API of stable versions such as <previous GA version>.
     > Only code written against a beta version such as <latest beta of this GA's minor> may be affected.
     ```
     Then list removed items using these phrasings:
     - `Below models do not exist in this release` — removed classes
     - `Below models are renamed` — renamed classes (`old -> new`)
     - `Below properties do not exist` — removed fields
     - `Below parameters do not exist` — removed method parameters
     - `<operation> does not exist in this release` — removed operations
     - Use fully qualified names (e.g., `azure.search.documents.models.VectorThreshold`).
