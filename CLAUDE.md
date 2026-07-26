# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## Project Overview

Three browser-only playgrounds for public life-sciences APIs (ClinicalTrials.gov v2,
openFDA, SEC EDGAR), used for Syngage lead research. Fork of
`hozefa-syngage/clinicalgovapi`. Active branch: `newctgov_test`.

## Architecture — zero build, one file per page

Each page is a **single self-contained HTML file**: one inline `<style>` block, one inline
`<script>` block, no modules, no framework, no `package.json`, no tests. The only external
dependency is Chart.js 4.4.1 from jsDelivr.

**Do not introduce a bundler, framework, or build step.** `render.yaml` publishes the repo
root directly (`env: static`, empty `buildCommand`), and the pages are expected to run when
opened straight from disk. Adding a build breaks both properties. If a build genuinely
becomes necessary, `render.yaml` must be updated in the same change.

There is real duplication between the two playgrounds (`syntaxHighlight`, `renderBarChart`,
`downloadFile`, `escapeHtml`, `renderEndpoint`, and the whole explorer shell). **This is
accepted, not an oversight** — deduplicating it requires modules and therefore a build step,
which costs more than the duplication does at this size.

## The `ENDPOINTS` registry is the extension point

Both playgrounds declare `const ENDPOINTS = {…}` (see
`clinicaltrials-api-playground.html:1050`). Each key holds `method`, `path`, `summary`,
`desc`, `useCase`, `presets[]`, and `params[]`. `renderEndpoint(key)`
(`clinicaltrials-api-playground.html:1205`) generates the entire UI from that object.

Adding an endpoint, a parameter, or a preset is a **data-only change** to `ENDPOINTS`. Do
not hand-write UI markup for a new endpoint. Sidebar entries live in the static
`.endpoint-item` list in the HTML body and must be added alongside the registry key.

Presets support an optional `group` field; `renderEndpoint` buckets chips by it.

## Treat the manual query-string building as deliberate

`buildUrl()` (`clinicaltrials-api-playground.html:1294`, especially the encoding at
`:1304-1317`) and `lfBuildUrl()` (`:2330`) assemble query strings by hand and then
selectively **un-escape** `%3A` → `:`, `%2C` → `,`, `%7C` → `|`, `%20` → `+`, rather than
using `URLSearchParams`. The in-code comment explains why: ClinicalTrials.gov was rejecting
percent-encoded `:` and `,` in `filter.*` / `aggFilters` values with
`Invalid format of aggregation-filter name:value pair`.

Verified against API v2.0.5 (2026-07-24): **both forms currently return 200** —
`aggFilters=phase%3A1` and `aggFilters=phase:1` both work, as do encoded and literal
`filter.advanced` brackets. So the original constraint appears to have been fixed upstream,
or was narrower than the comment suggests.

Do not refactor this to `URLSearchParams` just because the encoding now looks redundant.
The current code is known-good against the live API, the failure it guards against was real
at the time, and the payoff is nil. If you do change it, re-run the filtered presets against
the live API first — a regression here is silent (wrong results, not an error). The same
logic is duplicated in `openfda-edgar-playground.html:829`.

## Lead Finder (CT.gov playground only)

All Lead Finder code is prefixed `lf*` and lives at
`clinicaltrials-api-playground.html:2118-2634`. Flow:

1. `lfExtract(text)` (`:2172`) — regex-matches a pasted positioning document against the
   `LF_MODALITIES`, `LF_THERAPEUTIC_AREAS`, `LF_PHASES`, and `LF_SPONSOR_CLASSES`
   taxonomies, plus company size, geography, and target roles.
2. `lfBuildQueries(attrs)` (`:2233`) — picks whichever of modalities/therapeutic areas has
   **more** matches as the "anchor" dimension and emits one query per anchor (capped at 5).
   The opposite dimension is applied as a shared filter **only when it has exactly one
   item**, to avoid over-narrowing. A detected "Commercial" phase reserves one extra slot
   for a dedicated Phase 3 + `ACTIVE_NOT_RECRUITING` query (enrollment complete ≈ 6–12
   months from readout — the commercial-manufacturing decision window).
3. `lfExecuteAll(queries)` (`:2341`) — fans out with `Promise.all`.
4. `lfInferSignal(study)` (`:2357`) — classifies each trial into an outreach signal from
   status, phase, and post/update dates.

**Tuning lead quality means editing the `LF_*` taxonomies and the anchor logic**, not the
rendering functions. `PHASE_META` (`:2147`) is the shared phase reference used by the
attribute cards, results tables, and pipeline cards — update it in one place.

`LF_EXAMPLES` holds the built-in BIOVECTRA and Thermo Fisher positioning documents wired to
the example buttons.

## CORS: CT.gov is fine, EDGAR is not

ClinicalTrials.gov and openFDA permit direct browser calls.

SEC EDGAR requires a descriptive `User-Agent` per its Fair Access rules, which browsers
refuse to let scripts set — so direct calls fail with CORS errors or `403`. The
openFDA/EDGAR playground falls back to a chain of free public proxies
(`CORS_PROXIES`, `openfda-edgar-playground.html:896-899`: allorigins → codetabs →
cors.eu.org) behind a "Retry via CORS proxy (demo)" button.

**That chain is demo-only.** Those proxies are third-party and see the full request. Never
route credentials or sensitive data through them. Production EDGAR access must go through a
Syngage backend proxy setting `User-Agent: SyngageAI/1.0 ops@syngage.ai`. Do not remove the
warning banners that say so (`openfda-edgar-playground.html:781`).

## `clinicaltrials-api-playground-0.html` is a backup

It is the pre-Lead-Finder snapshot of the main playground, committed as a reference. The
landing page does not link to it. **Do not mirror edits into it** — leave it frozen. If it
ever stops being useful, delete it rather than maintaining it in parallel.

## Verification

There is no test suite; verification is manual in a browser.

```bash
python3 -m http.server 8000   # then open http://localhost:8000/
```

- The CT.gov playground header shows a live `API: v… · Data: …` string — if it reads
  `unreachable`, the `/version` fetch failed.
- Fastest end-to-end check of the URL encoding: `/studies` → "ADC Phase 1-2 (recruiting)"
  preset → Execute → expect HTTP 200 with results.
- Lead Finder: "BIOVECTRA" example button → "Extract & Find Leads" → expect attribute
  cards, an anchor banner, ~5 queries, and trial results.
- EDGAR endpoints are *expected* to fail on first attempt; that is the documented behavior,
  not a regression.

If tests are ever added, the pure functions (`lfExtract`, `lfBuildQueries`,
`lfGeoToCountries`, `lfInferSignal`) are the natural first targets — but they must be
extracted to a module first, which reintroduces the build-step question above.
