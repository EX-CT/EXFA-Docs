# 25 — Pyfa feature parity audit (post-migration)

Status: **current state audit** — the factual basis for the "match Pyfa" and "refactor the engine"
tracks (docs/08 roadmap). Written after the migration to EXFA-* (docs/24); supersedes the stale
numbers in docs/19's column headers (which still measure eve-dogma 2da8150).

## Verified state (evidence)

| backend | build | run | result |
|---|---|---|---|
| native CLI | EXFA-Engine v0.1.0 (`bf56fc2`) | release-CI 37404734 | every suite green, incl. `EXFA_PYFA_DATA` rows |
| wasip1 wasm | EXFA-Engine v0.1.0 | engine suites-wasm (wasmtime `--dir --env`) | all green: ext 239/239, ext_rpc 45/54 → **54/54** once pyfa presets carry `conversions` + dataset carries `descriptions` (see G1/G2; engine code already wired) |
| browser wasm | `exfa_wasm.wasm` on ex-ct.github.io/EXFA-App | pages run 37406678849, artifact `bench-suites-wasm-worker` | core 339/339 · effects 2378/2378 · graphs 192/192 · cap 150/150 · mutated 93/93 · formats 4779/4779 · batch 93/93 · ext 227/239 · ext_rpc 39/54 · sde 18/18 · price_inject 14/32 |

The 27 browser misses decompose into two real gaps and two **deliberate exclusions**:

### Deliberate exclusions (by design, not bugs)

| bench cases | why excluded | where the feature actually lives |
|---|---|---|
| `dpb_*`×6, `tpb_*`×6, `srch_*`×4 (jargon rows), `names_*` (GPL conversion part) | Pyfa-derived data is GPL-3.0 — it is *loaded at run time* (`--pyfa-data` / `EXFA_PYFA_DATA` / `pyfa_data_load`), never compiled into product builds. Docs/24 decided the product ships the EX-CT `presets.json` (CCP/LGPL-clean) instead. | native/wasip1 pass all of them when the pyfa presets file is attached (engine CI gate). The app ships `presets-web.json` (SDE-derived damage/target profiles + implant sets) at the UI layer. |
| `backup_*`×2 (`fits.backup`) | Formats RPCs (`fits.backup`, `eft_*`, `format_*`) live in `exfa-formats`, served by `exfa-cli` and by the separate `exfa_formats_wasm.wasm` — the engine wasm's method table deliberately excludes them. | web `backup/restore` + XML/EFT import-export work through the formats module (e2e `library-export-xml-eft`, `library-backup-restore` pass). |
| live multi-source price fetch (PRC-001 fuzzwork/evemarketdata/…) | offline-by-design: snapshot injection (`prices_load`, embedded `data/prices-*.json.gz`) replaces Pyfa's HTTP fetchers. | web pulls the newest `prices-*` release hourly and re-injects (App `fdaa3b1`); MCP `price.file`. |

### Real gaps — where the fix lives

| # | bench/UI symptom | root cause | fix lands in | size |
|---|---|---|---|---|
| G1 | `type_desc_*`×6: `type` returns `description: null` | dataset has no `descriptions` section — `sdepipe/build.py` never emits it (SDE `types.description` exists). Engine + codegen already wired (`HAS_DESCRIPTIONS`, `DESCRIPTIONS_GZ`). | EXFA-Data `sde/sdepipe/build.py` + next dataset revision (`dataset-3569502-r6`) | S |
| G2 | `names_*`×3: renamed items resolve to `null` | `pyfa_data_status` shows `conversions: 0` — the presets-pyfa generator (EXFA-Bench `oracle/`) doesn't extract `service/conversions` renamed items. Engine reads `conversions.items {old: new}` already. | EXFA-Bench oracle generator + regenerated presets-pyfa file | S |
| G3 | `srch_*` jargon empty in browser | engine jargon map only loads from the GPL pyfa file; EX-CT `search_aliases` (108 aliases in `presets.json` / `aliases/aliases.json`, LGPL) can't reach the engine. | EXFA-Engine: let `pyfa_data_load` (or a new `aliases_load`) accept the `exfa-search-aliases` / `exct-presets` `search_aliases` table → web injects it into wasm | S |
| G4 | `cimp_*` 2/6 | character implants vs fit implants (`implantSource`) — engine has per-request `implants` only | EXFA-Engine `calc` input + lookup | M |
| G5 | remaining `partial` rows (docs/19): alpha-clone skill caps, `sources`/`dependants` re-derivation, spool min/max exposure, `fits.export` variants, saved-profile libraries over RPC | see per-item evidence in docs/19 yaml | mostly EXFA-Engine; a few are UI-column work | M–L |

## UI parity (apps/web) — what "feels like pyfa" still means

The engine is one RPC away from parity for most rows; the **web UI** is the bigger surface. Audit of
`apps/web/src/ui/` against the pyfa workflow (docs/01 + inventory `UI-*`/`CHR-*`/`MKT-*` rows):

- **present**: market tree + meta tabs, fit browser, stats panes, price box (hourly refresh + status), import/export incl. DNA/multibuy, what-if variation swap, implant sets, custom profiles, EN/ZH.
- **to build for pyfa-grade UX** (click-driven, no full-screen pickers):
  1. slot-rail interaction parity: drop onto slots, double-click/右键 swap variation, per-module state toggle (online/offline/overheat), charge select on module
  2. drag & drop from market/search onto the fitting canvas
  3. character sheet view + **ESI login → skills import** (PKCE; injects `character`/`implants` into the fit request) — designed in `packages/` so the desktop app reuses it
  4. item compare pane (`item.compare` already in engine), type description panel (unblocked by G1)
  5. target profile / damage pattern pickers bound to the calc request (today presets exist; wiring them into the request like pyfa's top-bar selectors)

## Work order

1. **Close G1+G2** (data-side, no engine code): sdepipe `descriptions` + oracle `conversions` → engine/CI pass the last 9 wasip1 rows with zero engine changes.
2. **G3** small engine patch → browser `srch_*` via our LGPL aliases (pyfa jargon stays GPL-only).
3. **G4/G5** engine features + bench cases.
4. **Structural refactor** (docs/20): consolidate the 10 crates — merge `exfa-model` (356 LoC) + `exfa-sde` (233) + `exfa-capsim` (390) into `exfa-core`, keep `exfa-formats`/`exfa-optimizer`/`exfa-codegen` as peers, keep the two wasm shells as thin cdylibs (~40 LoC each, they must stay separate crates). Contract frozen: `calc`/`batch`/`serve-stdio` method tables + wasm exports; all bench suites must stay green.
5. Web UI items 1–5 above (pyfa-parity UX + ESI design).
6. Productization sweep: READMEs, stale references to eve-*, GH-only deploys; then the Cloudflare-vs-server deploy study.

## Non-goals

- Desktop app build (design for reuse only — ESI/prices modules in `packages/`).
- Archiving eve-* repos, `EXFA_DISPATCH_TOKEN` — user-owned steps.
