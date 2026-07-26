# API Playgrounds

Interactive, browser-only playgrounds for exploring the public life-sciences APIs that
feed Syngage lead research. No build step, no dependencies to install — each page is a
single self-contained HTML file.

## Pages

| File | What it does |
|---|---|
| `index.html` | Landing page linking to the two playgrounds |
| `clinicaltrials-api-playground.html` | ClinicalTrials.gov v2 API — **API Explorer** + **Lead Finder** tabs |
| `openfda-edgar-playground.html` | openFDA (drug labels, recalls, NDC) + SEC EDGAR (filings, XBRL, full-text search) |
| `clinicaltrials-api-playground-0.html` | Backup snapshot of the CT.gov playground taken before the Lead Finder was added. Not linked from the landing page. |

### ClinicalTrials.gov playground

**API Explorer** — pick an endpoint from the sidebar, fill parameters (or click a preset
chip), and Execute. Covers `/studies`, `/studies/{nctId}`, `/studies/metadata`,
`/studies/enums`, `/stats/*`, and `/version`. Responses render as syntax-highlighted JSON
plus purpose-built tables and charts.

**Lead Finder** — paste a seller positioning document, and the page extracts modalities,
therapeutic areas, phases, sponsor class, and geography from it, generates up to five
ClinicalTrials.gov queries, runs them in parallel, and classifies each resulting trial
into an outreach signal. Built-in example documents for BIOVECTRA and Thermo Fisher.

### openFDA / EDGAR playground

Same explorer shape. openFDA endpoints work directly from the browser. **SEC EDGAR
endpoints generally do not** — see below.

## Running locally

```bash
python3 -m http.server 8000
# open http://localhost:8000/
```

Opening the files directly via `file://` also works, but prefer the HTTP server so
relative links between the landing page and the playgrounds behave exactly as they do in
production.

The only external runtime dependency is Chart.js 4.4.1, loaded from jsDelivr, so an
internet connection is required for charts (and, of course, for the API calls themselves).

## Deployment

`render.yaml` declares a Render static site (`env: static`, empty build and start
commands, `staticPublicPath: ./`). The repo root is published as-is — which is only
possible because there is no build step. See `CLAUDE.md` before changing that.

## Known limitation: SEC EDGAR and CORS

SEC's Fair Access policy requires a descriptive `User-Agent` header on every request.
Browsers do not allow scripts to set `User-Agent`, so direct EDGAR calls from these pages
typically fail with a CORS error or `403`.

For demo exploration, the openFDA/EDGAR playground offers a **"Retry via CORS proxy
(demo)"** button that routes the request through a chain of free public proxies. This is
convenience only — those proxies are third-party, unreliable, and see the full request.

**In production, EDGAR calls must go through a Syngage backend proxy** that sets
`User-Agent: SyngageAI/1.0 ops@syngage.ai`. Never send anything sensitive through the
public proxy chain.
