# workspace-import-version-preservation

Pattern: `workspace-import-version-preservation`
pnpm version under test: `12.3.2`
Categories: `lockfile_format`, `tree_structure`, `version_constraints`
Schema version: `1.2`

## Feature exercised

This probe targets three behavioral changes introduced in pnpm 12.3.2:

1. **lockfile_format** (`pnpm import` version preservation) — pnpm 12.3.2
   changed `pnpm import` to preserve exact versions from the source npm
   lockfile instead of re-resolving them against the registry. The
   `pnpm-lock.yaml` in this probe was produced by running `pnpm import`
   from the accompanying `package-lock.json`. Every pinned version
   (`react@18.3.1`, `lodash@4.17.21`, `react-is@18.3.1`,
   `loose-envify@1.4.0`, `js-tokens@4.0.0`) is an imported exact pin,
   not a re-resolved version. Mend's `PnpmLockCollector` must read these
   exact versions from the lockfile — any drift signals a regression.

2. **tree_structure** (all-workspace import) — pnpm 12.3.2 fixed `pnpm
   import` to process every workspace project, not just the root. This
   probe has two workspace members (`packages/app`, `packages/lib`), each
   with its own importer entry in `pnpm-lock.yaml`. Mend must detect
   both importers as separate tree nodes; collapsing them into the root
   or skipping either member is the failure mode.

3. **version_constraints** (peer-resolution timing) — `packages/lib`
   declares `react-is@18.3.1` as a direct dep and peers on
   `react@^18.0.0`. pnpm 12.3.2 modified the timing of peer-dependency
   resolution during import. The `packages` section of the lockfile
   carries `peerDependencies` and `peerDependenciesMeta` for `react-is`.
   The expected tree encodes this relationship in `source_detail` for
   `react-is` so downstream comparison can assert correct peer handling.

## Project structure

```
.
├── package.json               # root (private, no direct deps)
├── package-lock.json          # npm source lockfile for pnpm import
├── pnpm-lock.yaml             # result of: pnpm import (v9 format)
├── pnpm-workspace.yaml        # declares packages/*
├── .whitesource               # Bucket A versioning pin
├── README.md
├── expected-tree.json
└── packages/
    ├── app/
    │   └── package.json       # @probe/app: react@18.3.1, @probe/lib workspace:*
    └── lib/
        └── package.json       # @probe/lib: lodash@4.17.21, react-is@18.3.1
                               #   peerDependencies: react@^18.0.0
```

## Expected dependency tree

- Root (`pnpm-import-version-preservation-probe`) has no direct
  registry deps; its workspace members appear as importers.
- `packages/app` importer direct deps: `@probe/lib` (local, link),
  `react@18.3.1` (registry, exact imported pin).
- `packages/lib` importer direct deps: `lodash@4.17.21` (registry,
  exact imported pin), `react-is@18.3.1` (registry, exact imported pin).
- `react` transitive chain: `react@18.3.1` → `loose-envify@1.4.0` →
  `js-tokens@4.0.0`.
- `react-is@18.3.1` peer dep on `react@^18.0.0` is recorded in
  `source_detail.peerDependencies`.
- All versions are exact pins from the imported npm lockfile —
  none were re-resolved by pnpm.

## Mend config

Bucket A — default-emit. pnpm has no dynamic version detection from the
manifest, so this probe pins the toolchain explicitly via `.whitesource`:

```json
{
  "scanSettings": {
    "configMode": "AUTO",
    "versioning": {
      "node": "20.18.1",
      "pnpm": "12.3.2"
    }
  }
}
```

- `pnpm` pinned to `12.3.2` — the version under test.
- `node` pinned to `20.18.1` — LTS compatible with pnpm 12.
- `configMode: "AUTO"` — no `whitesource.config` is shipped.

## Mend resolver notes

Mend's `PnpmLockCollector` detects `pnpm-lock.yaml` and routes to
`PnpmParserV9Impl` (lockfileVersion `9.0`). Key behaviors:

- The `importers` section drives workspace discovery. Each key
  (`packages/app`, `packages/lib`) is reported as a separate project
  scope. If the pre-12.3.2 behavior (root-only import) is present in
  the scanned lockfile, `packages/lib`'s importer entry would be
  missing and Mend would report zero deps for that workspace member.
- The `snapshots` section provides the actual resolved dependency lists
  (not `packages`). `PnpmParserV9Impl` reads `snapshots`, not
  `packages`, for dependency edges.
- Version preservation is verified by asserting that the versions in
  `expected-tree.json` exactly match the lockfile — there is no
  "expected re-resolved" variant.

## Probe metadata

```
probe_id:       workspace-import-version-preservation-20260904-023200
pm:             pnpm
pm_version:     12.3.2
schema_version: 1.2
categories:     lockfile_format, tree_structure, version_constraints
generated_at:   2026-09-04T02:32:00Z
pattern:        workspace-import-version-preservation
status:         generated
```
