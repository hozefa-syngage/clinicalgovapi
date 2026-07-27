# Lead Finder — Implementation Record

**Status:** built and verified, not deployed. **Date:** 2026-07-26.
**Branch:** `newctgov_test` (CTgov) · working tree (Syngage_Pipeline).
**Gate before production port:** Shreya's validation round.

This document covers two rounds of work on the CT.gov Lead Finder:

- **Phase A** (shipped earlier, commit `4f0ea89`) — turned a flat list of trials into a ranked
  list of sponsor accounts.
- **Phase A2 + Phase B** (this round, uncommitted) — made the ranking depend on *who is
  selling*, and added an optional LLM re-rank served from the existing Modal app.

---

## 1. Why this round happened

Phase A worked mechanically — 5 queries, 332 deduped trials, 176 distinct sponsors, ranked
0–100 with an explainable breakdown. But the output was wrong for the seller it was tested
against.

For **BIOVECTRA**, a CDMO, the top 10 was:

```
100  24 trials  AstraZeneca                     Enrollment complete
100  13 trials  Janssen Research & Development  Enrollment complete
 98   9 trials  Pfizer                          Enrollment complete
 97   7 trials  BeOne Medicines                 Enrollment complete
 92   3 trials  Genmab                          Enrollment complete
 92  10 trials  Bristol-Myers Squibb            Enrollment complete
 92  14 trials  Hoffmann-La Roche               Enrollment complete
 92   8 trials  Novartis Pharmaceuticals        Enrollment complete
 91   7 trials  Merck Sharp & Dohme LLC         Enrollment complete
 90   6 trials  Regeneron Pharmaceuticals       Enrollment complete
```

Three problems, one root cause.

**The root cause: 70 of 100 points rewarded sponsor size.**

| Dimension | Max | Why it favoured size |
|---|---|---|
| Anchor match | 25 | A sponsor running trials in every modality matches more queries for reasons unrelated to fit |
| Phase fit | 20 | Maxed on Phase 3, which only large sponsors run at volume |
| Portfolio depth | 15 | Monotonic in trial count |
| Scale | 10 | Enrollment count |
| **Total size-correlated** | **70** | |
| Recency | 20 | size-neutral |
| Sponsor class | 10 | size-neutral |

That is backwards for a CDMO. AstraZeneca and Roche manufacture in-house and buy through
standing framework agreements. The buyer is the Phase 1–2 biotech with two trials and no plant.

But it is *correct* for a different seller. The transcript's Alphin → Pfizer example is a drug
developer seeking a partnership, where large-cap **is** the target.

**So sponsor size is not universally good or bad — its value depends on the seller's
archetype.** That reframing is the whole of Phase A2, and it operationalises the transcript's
central point ("think from the customer's point of view") using the five archetypes that
already exist in [`Syngage_UI/lib/archetypes.ts`](../Syngage_UI/lib/archetypes.ts).

Two secondary symptoms fell out of the same cause:

- **Score compression** — 8 of the top 10 sat at 90+, with ties at 100.
- **Signal homogeneity** — all 10 read "Enrollment complete", so the demand strip printed the
  same sentence ten times and differentiated nothing.

---

## 2. Result

Same 332 trials, same 176 sponsors. Only the seller archetype changes:

| | Top 10 for **BIOVECTRA (CDMO)** | Top 10 for **drug developer** |
|---|---|---|
| Before A2 | AstraZeneca, Pfizer, Roche, Novartis… — **10/10 large-cap** | identical |
| After A2 | BeOne, Chia Tai Tianqing, Genmab, Shanghai Henlius… — **0/10 large-cap** | AstraZeneca 100, Janssen 100 — **7/10 large-cap** |

| Metric | Before A2 | After A2 |
|---|---|---|
| Large-cap in top 10 (CDMO seller) | 10/10 | 0/10 |
| Score range | 43–100 | 62–95 |
| Top 10 clustered at 90+ | 8/10 | 4/10 |
| Distinct signals in top 10 | 1 | 3 |
| Size tiers across 176 sponsors | — | emerging 127 · mid 38 · large 11 |

Changing the archetype dropdown re-ranks instantly with **no new API calls** — the retrieved
trials are held in `LF_LAST` precisely so the before/after comparison uses identical data.

---

## 3. Phase A2 — archetype-conditional scoring

All changes are inside the existing inline `<script>` in
[`clinicaltrials-api-playground.html`](clinicaltrials-api-playground.html). No bundler, no
framework, no build step — `render.yaml` (`env: static`, empty `buildCommand`) and
open-from-disk both still work. `clinicaltrials-api-playground-0.html` stays frozen.

### 3.1 Archetype detection — `LF_ARCHETYPES`, `lfDetectArchetype`

A keyword taxonomy in the same shape as `LF_MODALITIES`. The five `value` strings are copied
verbatim from `archetypes.ts` so this ports to the pipeline without translation.

Detection counts **distinct patterns matched**, not occurrences, so one word repeated eight
times cannot outvote six independent signals.

Two deliberate choices worth reviewing:

- **`drug_device_developer` uses phrases, not single words.** Bare `/develop/` or `/\bIND\b/`
  match every CDMO document too — BIOVECTRA's own text contains *"companies developing small
  molecule drugs"* and *"from IND through commercial manufacturing"*. The patterns are
  `our lead candidate`, `we are developing`, `clinical-stage`, `first-in-class`, `IND-enabling`.
- **The fallback is `not_life_sciences`, which scores size neutrally.** A wrong archetype guess
  would *invert* the ranking, so "no opinion" is the safe default rather than picking the most
  common archetype.

Measured on the built-in examples: BIOVECTRA → CDMO (7 signals); Thermo Fisher →
technology provider (6 signals). Thermo is the harder case — it mentions "CRO" three times but
leads with instruments/reagents/consumables/software, and the distinct-pattern count resolves
it 6-to-3 in favour of tech provider.

**Override:** a `<select>` sits in the archetype card. Regex over prose will misfire, and the
user knows better. Changing it re-scores and re-renders without re-querying.

### 3.2 Sponsor size — `lfSizeTier`

CT.gov has no company-size field. This is a trial-footprint proxy:

| Tier | Rule |
|---|---|
| `large` | `trialCount >= 13`, or `maxEnrollment >= 1000` with a Phase 3/4 |
| `mid` | `trialCount >= 5`, or has a Phase 3/4 |
| `emerging` | everything else |

**This is the weakest part of the design and the labels say so.** It measures footprint *within
the retrieved candidate pool*, not globally. Observed in real runs:

- **Bayer reads `emerging`** because only 2 of its trials matched these queries.
- **Pfizer reads `mid-cap`** with 9 matching trials.
- A well-funded 20-person biotech running one large Phase 3 would read `mid`.

The badge tooltip states "Inferred from this sponsor's footprint in these results — not
headcount or revenue." Widening the pool via pagination would sharpen it; that is a follow-up,
not a v1 fix.

### 3.3 Rebalanced scoring — still 0–100, still explainable

| Dimension | Was | Now | Rationale |
|---|---|---|---|
| Anchor match | 25 | **20** | Partly a size proxy in disguise |
| Phase fit | 20 | 20 | unchanged |
| Recency | 20 | 20 | unchanged, size-neutral |
| **Buyer fit** (size tier × archetype) | — | **25** | new; replaces Portfolio depth + Scale |
| Sponsor class | 10 | 10 | unchanged |
| Portfolio depth | 15 | **5** | 3 trials genuinely beats 1, but should not decide the ranking |
| Scale | 10 | — | removed; folded into Buyer fit |

`LF_ARCHETYPE_FIT` is the table that does the work:

| Archetype | emerging | mid | large |
|---|---|---|---|
| `commercial_pharma_manufacturing_service_provider` | 25 | 20 | **8** |
| `technology_service_provider` | 18 | **25** | 14 |
| `drug_device_developer` | 12 | 20 | **25** |
| `investor` | **25** | 16 | 5 |
| `not_life_sciences` | 15 | 15 | 15 |

Each entry carries a `why` string that renders into the existing "Why this score" breakdown, so
a card explains *why size counted the way it did* rather than just showing a number.

**These numbers are a first calibration, not a measurement.** They encode a plausible BD
heuristic. Shreya's round is what tells us whether `large: 8` for a CDMO seller is right or too
harsh. Expect to tune them — that is the intended use of the table.

### 3.4 Tie-break changed to recency

Integer dimensions produce constant ties — **32 of the top 40 adjacent pairs are tied**. The
original fallback was `trialCount`, which would have quietly reinstated the exact bias Buyer fit
exists to remove: the bigger sponsor wins every close call.

Now: score → `newestLastUpdate` desc → `trialCount`. Recency is size-neutral and matches the
"what just changed" framing the feed is meant to have.

### 3.5 Archetype-aware top signal

`lfTopSignal` now prefers a signal whose `LF_DEMAND_MAP[type].activates` includes the seller's
archetype, falling back to the existing `LF_SIGNAL_PRIORITY` order.

**Honest scope note:** this matters for `investor` and `drug_device_developer`, where
`activates` is narrow. For CDMO and tech-provider sellers all five signals activate them, so the
archetype pass is an identity function. **The observed signal diversity gain (1 → 3 types) came
from §3.3, not from this change** — signal homogeneity was a symptom of the mega-pharma bias,
not an independent bug. This change is still correct, just less load-bearing than it looks.

---

## 4. Phase B — LLM re-rank via Modal

The playground is a static page with nowhere safe to hold an Anthropic key. Rather than put the
key in `localStorage`, the key stays server-side on the existing `syngage-pipeline` Modal app
(which already loads the `syngage-anthropic` secret) and the page presents a scoped token.

New route on the **existing `web()` ASGI app** — no new Modal function, no `campaign_steps`
row, no `.spawn()`. One synchronous call, deliberately outside the pipeline state machine.

### 4.1 `POST /v1/playground/ctgov/rerank`

[`Syngage_Pipeline/api/routes_playground.py`](../Syngage_Pipeline/api/routes_playground.py)

- **Auth:** bearer token, `hmac.compare_digest`, following the `routes_hooks.py` precedent for
  routes that legitimately cannot present the two-token JWT. Header, not query param — query
  strings land in access logs. **Fails closed with 503 if `PLAYGROUND_API_TOKEN` is unset**; an
  unset secret must never mean "open".
- **Body:** top ~25 compact sponsor records (name, class, size tier, trial count, phases,
  signals, deterministic score, ≤3 NCT IDs + titles). Never raw study objects.
- **Model:** `claude-haiku-4-5-20251001` by default — a ranking pass over a small payload, the
  same tier Step 1a uses. `{"model": "sonnet"}` opts into `claude-sonnet-4-6`.
- **Prompt:** carries a per-archetype rubric restating what "a good lead" means for this seller,
  mirroring `build_scoring_guide`. Without it the model ranks on brand recognition and lands
  straight back on mega-pharma. The deterministic score is passed as "a hint you may overturn,
  not ground truth".
- **Reuses** `call_claude` / `parse_claude_json` from `pipeline/utils.py`, so retry, refusal
  handling and cost accounting come free.
- **Caps:** rejects >40 sponsors, bounds `max_tokens`.

### 4.2 The hallucination guard — `_validate_against_submitted`

Every returned sponsor and NCT ID is checked against what was submitted. NCT IDs must belong to
**that** sponsor, so the model cannot borrow a real trial from another company to support a
claim. Dropped counts are reported to the UI and surfaced to the user.

**This caught a real problem on the first live call.** Haiku emitted `sovereignName` instead of
`sponsorName` for one entry — a key-name slip, not a hallucination. The strict check discarded a
legitimate lead (14 returned instead of 15).

The fix preserves the security property rather than loosening it: if the name key is missing,
recover the sponsor from the cited NCT IDs — **if and only if every one of them resolves to
exactly one submitted sponsor**, which proves the entry is grounded in data we sent. Ambiguous
citations spanning two sponsors, or unrecognised ones, still drop. Verified by three dedicated
tests including an explicit "recovery must not become a backdoor" case.

### 4.3 CORS — the app's first

There was **no CORS middleware anywhere** in the FastAPI app; the Next.js frontend calls Modal
server-side, so it never needed any.

[`api/app.py`](../Syngage_Pipeline/api/app.py) now adds `CORSMiddleware` with:

- an **explicit origin allowlist** (`localhost:8000`, `127.0.0.1:8000`, plus
  `PLAYGROUND_CORS_ORIGINS`). `allow_origins=["*"]` would expose campaigns, prospects and runs
  to any web page on the internet.
- `allow_credentials=False` — auth is a bearer header, not a cookie, so credentialed CORS buys
  nothing and only widens the surface.
- methods limited to `POST, OPTIONS`.

**Consequence:** `file://` sends `Origin: null` and is refused (verified: 400). Phase B requires
serving the page over `http://localhost:8000`. Phase A2 still works fine from disk.

### 4.4 Security posture — internal only

The playground token is readable by anyone with devtools open on the page. That is meaningfully
better than exposing the Anthropic key — it is scoped to one endpoint, does nothing but rank
sponsors, and is revocable on its own — but it is **not a customer-facing design**.

- Token entered at runtime, stored in `localStorage`, never committed.
- A warning banner in the settings panel says so, following the precedent of the CORS-proxy
  warnings in `openfda-edgar-playground.html:781`.
- Deploy to `--env=staging`, not prod.
- Replace with per-user auth before anyone outside the team sees it.

**Watch item:** `web()` has `timeout=60` (`modal_app.py:71`), shared with every other route.
Haiku returns in ~10s; if the Sonnet toggle ever pushes past 60s, raise it deliberately.

---

## 5. Files changed

**`CTgov/clinicaltrials-api-playground.html`** — new `LF_ARCHETYPES`, `lfDetectArchetype`,
`lfArchetypeLabel`, `LF_SIZE_TIERS`, `lfSizeTier`, `LF_ARCHETYPE_FIT`, `LF_AI_KEYS`,
`lfAiConfig`, `lfAiPayload`, `lfAiRerank`, `lfAiRenderPanel`, `lfSetArchetype`, `LF_LAST`;
edits to `lfExtract`, `lfScoreSponsor`, `lfTopSignal`, `lfRollupSponsors`, `lfRenderAttrs`,
`lfRenderSponsorCard`, `lfRenderResults`, `lfRun`; CSS and settings markup.

**`Syngage_Pipeline/api/routes_playground.py`** — new.
**`Syngage_Pipeline/api/app.py`** — `CORSMiddleware` + router.
**`Syngage_Pipeline/CLAUDE.md`** — documents the endpoint, the secret, and the CORS rationale.

Reused rather than rewritten: `call_claude` / `parse_claude_json` / `build_scoring_guide`
(`pipeline/utils.py`), the `_verify_token` pattern (`api/routes_hooks.py`), `PHASE_META`,
`phasePillHTML`, `LF_DEMAND_MAP`, `lfBuildUrl` (encoding untouched — see `CLAUDE.md`).

---

## 6. Verification

**71 automated checks pass** — 43 playground + 28 endpoint — plus real-browser and real-model
runs. There is no test suite in the repo; these live in the session scratchpad.

### Automated

```bash
# 43 checks — live CT.gov API, archetype flip, score integrity, rendering
node <scratchpad>/verify.mjs

# 28 checks — auth, CORS, hallucination guard (Claude stubbed, no API cost)
cd Syngage_Pipeline && .venv/bin/python <scratchpad>/test_playground.py
```

Notable assertions:

- `CDMO seller: top 10 is NOT dominated by large-cap` — 0/10 (was 10/10)
- `drug developer seller: large-cap climbs back` — 7/10 vs 0/10
- `dimension maxima sum to 100` — 20 + 20 + 20 + 25 + 10 + 5
- `ties are broken by recency, not portfolio size` — 32 tied pairs, all recency-ordered
- `NCT borrowed from a DIFFERENT sponsor is dropped`
- `nameless entry with unknown NCTs still dropped` / `ambiguous NCT recovery drops instead of guessing`
- `unset secret fails CLOSED (503, not open)`
- `preflight from an unlisted origin refused`

### Manual / real-browser (headless Chrome, driving the actual page)

| Check | Result |
|---|---|
| BIOVECTRA renders 20 cards, CDMO detected | ✅ emerging/mid lead |
| Archetype override flips ranking without re-query | ✅ AstraZeneca/Janssen return at 100 |
| Thermo Fisher → tech provider, TA-anchored | ✅ mid-cap favoured |
| Encoding regression (ADC preset → Execute) | ✅ HTTP 200, real ADC trials, `%3A`/`%2C` un-escaped, brackets encoded |
| No token configured | ✅ 20 cards, no re-rank bar, no AI panel |
| End-to-end AI re-rank | ✅ 15 leads, $0.0095, zero drops, NCT evidence links |
| `Origin: null` (`file://`) rejected | ✅ 400 |
| Wrong token over real HTTP | ✅ 401 |

Two live Anthropic calls were made during verification (~$0.019 total) using the key in
`Syngage_Pipeline/.env`.

---

## 7. Known limitations

1. **The size proxy is footprint-within-results, not company size.** Bayer reads `emerging`.
   Labelled as inferred; pagination would sharpen it.
2. **`LF_ARCHETYPE_FIT` numbers are unvalidated judgement.** They are the first thing to tune
   after Shreya's round.
3. **Archetype detection is regex over prose and will misfire.** The override select is the
   mitigation, not a nicety.
4. **Recency-first sampling is not relevance-first.** `sort=LastUpdatePostDate:desc` with
   `pageSize=100` means a query matching 288 trials still shows the 100 most recently updated.
5. **Sponsor-name collapse is exact-match only.** "Pfizer"/"Pfizer Inc." merge; "Pfizer"/"Pfizer
   Ireland Pharmaceuticals" do not. Deliberate — fuzzy merging silently fuses distinct
   subsidiaries, which is worse than a visible duplicate.
6. **Adding CORS widens the Modal app's surface.** The allowlist must stay explicit.
7. **Contact discovery is still unsolved.** Phase A2 gets to the right *company*; Apollo gets to
   the right *person*. That bridge only closes on the production port — say so when demoing.

---

## 8. Not done — required before this runs anywhere but a laptop

1. **Create the secret** (needs your Modal account):
   ```bash
   modal secret create syngage-playground PLAYGROUND_API_TOKEN=$(openssl rand -hex 24)
   # then add modal.Secret.from_name("syngage-playground") to `secrets` in modal_app.py
   ```
   Lower-friction alternative: add `PLAYGROUND_API_TOKEN` to the existing `syngage-anthropic`
   secret — no code change needed.
2. **Deploy** — `modal deploy modal_app.py --env=staging`.
3. **Commit** — both repos are uncommitted. Nothing was committed or pushed.
4. **Render** — not required; Shreya runs locally. If that changes, `master` is a clean
   fast-forward ancestor of `newctgov_test`, and `render.yaml` does not pin a branch (that is a
   Render dashboard setting).

---

## 9. Port path to production

Unchanged from the approved plan, and Phase A2 does not complicate it — the decision functions
(`lfExtract`, `lfDetectArchetype`, `lfSizeTier`, `lfScoreSponsor`, `lfBuildQueries`,
`lfGeoToCountries`, `lfInferSignal`, `lfNormalizeSponsor`, `lfRollupSponsors`) are all pure and
DOM-free, so they port to `pipeline/*.py` as `run(args) -> dict` library functions without
rewriting. Rendering functions are browser-only and get discarded.

- **Step 1c** — `pipeline/search_ctgov_trials.py` + `workers/step1c_trial_leads.py`; migration
  adding `'1c'` to `chk_lead_source`; `/step1c/start` in `api/routes_campaigns.py`; a third
  sub-tab in the campaign Leads view. Reuses `enrich_apollo_contacts`, `score_leads`,
  `campaign_steps` anchoring, and the Realtime progress bar.
- **Gap to close first** — Step 0's `intelligence` has `therapeutic_areas` but **no `modality`**
  (`analyze_company_intelligence.py:93`). The strongest CT.gov anchor has no source field, so
  queries would still be built from regex rather than structured data. The seller archetype is
  already stored, so Phase A2's key input needs no new plumbing.
- **Option D (regulatory signals feed)** — `regulatory_signals` + `signal_watches` are fully
  designed in `supabase_schema_plan_v2.md:407-470` but never built. `lfInferSignal` +
  `LF_DEMAND_MAP` become its classification layer once validated here.

---

## 10. What to judge this on

Not a metric. **Shreya runs 3–5 real pharma positioning documents through it and says whether
the top 20 are companies she would actually contact.** If the answer is no, the first thing to
change is the `LF_ARCHETYPE_FIT` table, then the size-tier thresholds — both are single tables,
edited in one place, with no other code depending on their values.
