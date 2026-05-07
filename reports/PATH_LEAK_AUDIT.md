# PATH_LEAK_AUDIT

**Scope:** Personal `C:\Users\fonzi`, NBC OneDrive paths, internal D: drive paths, USERPROFILE templates that leak NBC folder structure, across 3 repos.
**Branch base:** Stormgauge `staging`, Stormgrid `main`, Hub `main`.
**Method:** `git ls-files` per repo → grep tracked files only (untracked working-tree files such as `.tmp_*` scratch scripts and recovery copies are ignored — they don't ship).
**Date:** 2026-05-07.

---

## Headline

| Repo | Tracked files with leaks | Lines | Ships to public domain? |
|---|---|---|---|
| Stormgauge (`pluvio-stormgauge`) | 5 | 14 | **YES** — staging deploys to stormgauge.pluviometrics.com.au |
| Stormgrid (`stormgrid`) | 0 | 0 | n/a — clean |
| Hub (`pluviometrics-hub`) | 0 | 0 | n/a — clean |

Only Stormgauge has tracked path leaks. Stormgrid and Hub passed.

---

## Stormgauge — tracked findings

### `index.html` (deployed file) — 8 lines

| Line | Pattern found | Severity | Visibility |
|---|---|---|---|
| 722 | `<div id="fwLocalFolderPath">%USERPROFILE%\OneDrive - Northern Beaches Council\Stormwater Engineering - General\Program</div>` | High | Rendered in DOM, visible to any user who opens the Future Works tab. Discloses NBC's internal SharePoint folder structure. |
| 726 | `<div id="fwLocalWorkbookPath">%USERPROFILE%\OneDrive - Northern Beaches Council\Stormwater Engineering - General\Program\2027-2031 CAPEX Project Brief - Program of Works Details - E&R - Rolling.xlsx</div>` | High | Same — and discloses an internal CAPEX workbook filename. |
| 1371 | `const FUTURE_WORKS_LOCAL_FOLDER_TEMPLATE = '%USERPROFILE%\\OneDrive - Northern Beaches Council\\Stormwater Engineering - General\\Program';` | High | JS constant used to populate the above DOM nodes. |
| 5188 | `const CRITICAL_DEFAULT_SOURCE_PATH = 'D:\\Packaging\\data\\assets_with_coords.csv';` | Medium | Hardcoded NBC engineering workstation path. Not personally identifying but signals internal infrastructure. |
| 5189 | `const CRITICAL_DEFAULT_COMBINED_PATH = 'D:\\Packaging\\data\\combined_assets.csv';` | Medium | Same. |
| 5190 | `const CRITICAL_DEFAULT_INSPECTION_PATH = 'D:\\Packaging\\data\\critical_asset_inspections.xlsx';` | Medium | Same. |

### `Superseeded/before-blank-screen-fix-20260430/index.html` — 3 lines (legacy snapshot)

Lines 717, 721, 1366 — same `%USERPROFILE%\OneDrive - Northern Beaches Council\...` template strings as live `index.html`. This file is tracked and ships, but is not routed to.

### `Superseeded/direct-calculator-route-before-fix/index.html` — 6 lines (legacy snapshot)

Lines 717, 721, 1366 — `%USERPROFILE%` template strings.
Lines 5183–5185 — `D:\Packaging\data\` constants.

### `data/pluviometrics_ifd_table.json` — 1 line (build metadata, ships)

Line 3: `"source_input": "C:\\Users\\fonzi\\Weather App Folder\\data\\pluviometrics_rainfall_stations.json"` — provenance metadata exposing the build operator's local path. Not security-critical but discloses the maintainer's working directory.

---

## Stormgrid — clean (tracked)

`git grep` against the audit branch returns zero hits in tracked files for any of the patterns. The `index.html` shell, all `src/*.js` modules, and all shipped `data/*.json` files are free of personal/internal paths.

Note: build-only scripts and one local manifest do contain author paths — they are listed below for completeness but **do not ship** (Cloudflare Pages serves only what is in branch root, and these are operator tooling):

| File | Line | Path | Ships? |
|---|---|---|---|
| `scripts/build_asset_snapshot.py` | 25 | `D:/Packaging/data/assets_with_coords.csv` (default arg in CLI help) | No — Python build script, not deployed |
| `scripts/build_catchment_ifd.py` | 18 | `C:\\Users\\fonzi\\Weather App Folder` (docstring example) | No — same |
| `data/catchments/manifest.json` | 7, 70–73 | `C:\Users\fonzi\Weather App Folder\Assets\Catchments\...` | **Yes — this is in `data/` and ships.** Build-time provenance only; not loaded by the runtime, but visible to anyone who opens the URL. |

The `manifest.json` provenance disclosure is low-severity (no credentials, no DOM rendering) but should be sanitised in the cleanup phase.

---

## Hub — clean

Zero tracked path leaks. The `xargs grep` initially flagged `index.html` due to a pattern false-positive; precise re-check returned no matches. Hub ships only static HTML, CSS, logos.

---

## Summary by severity

| Severity | Count | Description |
|---|---|---|
| **High** (publicly visible NBC internal paths) | 4 lines | Stormgauge `index.html` 722, 726, 1371; one in each Superseeded copy |
| **Medium** (D:\ infra path constants in shipped JS) | 6 lines | Stormgauge `index.html` 5188–5190 + 3 in Superseeded copy |
| **Low** (build provenance metadata in shipped JSON) | 5 lines | `data/pluviometrics_ifd_table.json:3`, `stormgrid/data/catchments/manifest.json` ×4 |

Total: **15 lines across 5 tracked files**, all in Stormgauge and Stormgrid build artifacts. Hub is clean.

---

## Recommended next fixes (NOT applied — listed for approval)

1. **Stormgauge `index.html` Future Works tab (lines 722, 726, 1371)** — replace the hardcoded NBC SharePoint path with a configurable template loaded from `window.STORMGAUGE_CONFIG.futureWorksFolderTemplate`, or remove the path display entirely (it's NBC-internal documentation that has no value to non-NBC users). Risk: low — pure presentation, no calculation impact.
2. **Stormgauge `index.html` Critical Assets defaults (lines 5188–5190)** — convert to relative or empty-string defaults; let users set their own via the existing UI control. Risk: low — these are fallback paths only used if the user hasn't entered one.
3. **Stormgauge `data/pluviometrics_ifd_table.json` line 3** — strip the `source_input` field at build time (`scripts/enrich_ifd.py` should write `"source_input": "(local build)"` or omit). Risk: zero — metadata only.
4. **Stormgrid `data/catchments/manifest.json` lines 7, 70–73** — replace absolute paths with repo-relative paths in `scripts/extract_catchments.py` (or whichever script writes this manifest). Risk: zero — provenance metadata only.
5. **Stormgauge `Superseeded/`** — out of scope for path-leak fix; recommend a separate housekeeping pass to remove `Superseeded/` from the deployed branch entirely (it's 1.6 MB of dead HTML).

No code changes have been made.
