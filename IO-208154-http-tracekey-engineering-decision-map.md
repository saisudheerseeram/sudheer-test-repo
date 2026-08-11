# IO-208154 · HTTP Trace-Key Engineering Decision Map

Fresh validation of all ten proposals against clean current-main clones, the linked Jira/Confluence material, and the evidence workbook.

---

## 1. Executive Summary

**Bottom line:** Ship the small correctness fixes first, but do not implement the ten items as ten isolated PRs. Changes 5 and 6 require one batch-aware framework; Change 8 is a shared-evaluator problem — one confirmed silent-failure defect (D2) plus a strong wrong-record-shape hypothesis pending production samples (see 4.2); and Change 10 should expose reasons without generating a synthetic key. The largest operational risk is package skew across design-time, workers, and preview.

| Metric | Value |
|---|---:|
| Changes mapped to code | 10 / 10 |
| Current defects found (D1–D6) | 6 |
| Repositories audited (clean main clones) | 10 |
| Repositories in the deployment path | 8 |
| Product source files changed (read-only audit) | 0 |

---

## 2. Production Evidence (Rechecked)

| Category | No-key logs (%) | Duplicate logs (%) | Interpretation |
|---|---:|---:|---|
| Legacy HTTP · no template | 84.37% | 0.33% | Coverage is the primary problem |
| Legacy HTTP · template | 0.24% | 1.50% | Templates improve coverage but can increase collisions |
| CF2 HTTP · no template | 77.34% | 10.79% | Both missing and duplicate risk are high |
| CF2 HTTP · template | 20.14% | 9.50% | Confirms the parity gap targeted by Change 8 |
| GraphQL · no template | 7.42% | 1.39% | Residual issue smaller than generic HTTP |

*Source: Google Sheet "Summary w/ config from logs". These are log-level percentages; the Confluence 3.9% → 21.5% statement is a job-level duplicate metric and should not be compared directly.*

### Data Provenance and Known Limitations

| Item | Detail |
|---|---|
| Data window | `FLOW_LOG_SUMMARY` covers only ~25 days (Mar 22 – Apr 15) and is **stale since April 15** — a data-pipeline ticket should be raised before setting new KPI baselines (per the source Confluence page) |
| Denominator inflation | 529 internal "Worker Health Check Export" exports (hitting `system/v1/timestamp` every ~20 min) add ~2.4M no-key logs; they must be **excluded from trace-key KPI denominators** |
| Metric level | All percentages above are log-level, extracted from the evidence workbook; job-level metrics from Confluence are not directly comparable |
| Still needed | Exact extraction date, query filters and per-cohort denominators should be attached to the workbook before these numbers are used as rollout baselines |

---

## 3. Defects Found During the Audit (D1–D6)

D1–D6 are **not** part of the ten proposed changes. They are existing bugs discovered while verifying those proposals. Each needs its own Jira ticket.

| Defect | Location & issue | Fix summary | What the fix solves |
|---|---|---|---|
| **D1** | Producer returns `templates` (patternFinder.js:113/143); consumer reads `template` (main.js:47–52) | Consumer accepts both keys; producer emits `template` (keep `templates` one release) | NetSuite (`{{Images.image_record_id}}`) and Zendesk (`{{name}}-{{email}}`) map templates start working; unblocks template-based Change 1 profiles (field-only profiles ship independently) |
| **D2** | evalExpressionSync wrapper passed to processTraceKey (main.js:201–211); failures collapse to null | Unwrap wrapper; return null on error. Telemetry: aggregate **once per flow/template/reason** with a structured `logName` — never per-record warnings. **Never log raw template strings** (they can embed sensitive values) — log a template fingerprint (hash) plus the error code | Separates "template failed" from "field empty"; foundation for Change 10 and Change 8 attribution |
| **D3** | Map lookup uses `assistant \|\| type`; CF2 connections often have no assistant (connection.js:5507–5517) | Resolve legacyId at pattern-build time when `_httpConnectorId` present | CF2 connectors inherit curated map profiles; attacks 77.34% CF2 no-template cohort |
| **D4** | Preview calls getTraceKey without traceKeyPattern (responseTransformUtil.js:11) | Not a one-file fix: the pattern **originates** at design time (em-util → flow-management `getTraceKeyPattern`); the preview API contract must carry it into endpoint-service (processorService → responseTransformUtil), or endpoint-service must compute it from exportDoc + connection. Test **both** configured-template and no-template preview paths | Preview matches production for the **no-template** path (an explicit template short-circuits before pattern fallback — main.js:41–43 — so D4 does not affect the with-template cohort) |
| **D5** | flowCache is unbounded array (main.js:21, 56–59); undercounts missing-key telemetry | Replace with a **bounded LRU** (defined max entries, evict least-recently-seen) or TTL-based cache — not merely a size-capped Set, which needs an eviction policy anyway; keep once-per-flow logging | Bounds worker memory; does **not** cause duplicates — only undercounts missing-key logs |
| **D6** | Consumer version skew (8.0.1–8.0.19 across repos) | Publish tracekey-common → tracekey; bump identical pins every phase | Prevents design-time, preview, and production running different algorithms |

---

## 4. The Ten Proposed Changes

| # | Title | Status | Size | Phase | Primary owner | Primary locations |
|---:|---|---|---|---:|---|---|
| 1 | Expand assistant map | Proceed (templates need D1) | M | 2 | integrator-common-util | map.js, patternFinder.js, main.js |
| 2 | Detect Shopify/Amazon HTTP | Rework | M | 2 | integrator-common-util | patternFinder.js, connection.js, map.js |
| 3 | Extract Handlebars from URI | Proceed | S | 1 | integrator-common-util | patternFinder.js:274–311 |
| 4 | Widen identifier fallbacks | HTTP-scope | M | 4 | integrator-common-util | patternFinder.js:274–311 |
| 5 | Response-batch inference | Framework | XL | 4 | common-util + adaptor + flow-execution + endpoint/UI | tracekey-common, exportDataConverter, abstractExport, injectTraceKeys (flow-execution), endpoint |
| 6 | Pre-stamp uniqueness gate | With Change 5 | L | 4 | common-util + adaptor + flow-execution + endpoint/UI | tracekey-common, adaptor, injectTraceKeys (flow-execution), endpoint, UI |
| 7 | GraphQL support | Proceed | M | 3 | integrator-common-util | connection.js, patternFinder.js, main.js |
| 8 | CF2/template divergence | Needs samples | M | 3 | common-util + endpoint | main.js, baseHBDelegate.js, endpoint |
| 9 | Reduce noisy URI segments | Proceed | S | 1 | integrator-common-util | patternFinder.js:286–309 |
| 10 | Explicit no-reliable-key state | Proceed | L | 4 | common-util + adaptor + FE + endpoint/UI | main.js, abstractAdaptor, qgmw, endpoint, UI |

*Size legend (tracker scale S/M/L/XL, one value per change): size = total effort — research, validation, tests, rollout — not diff size. Final sizing after two review rounds: C3, C9 = S (small, ready code changes); C1, C2, C4, C7, C8 = M (real research/sampling/validation effort beyond small diffs); C6, C10 = L (multi-repo contract and propagation work); C5 = XL (new cross-repo batch-inference framework).*

### Key Caveats

| Change | Caveat |
|---:|---|
| 1 | Map entry runs only when `connection.assistant \|\| connection.type` matches |
| 4 | "Four-field dictionary" is actually 18 case variants over 5 base names on current main |
| 5 / 6 | Page-scoped and deterministic; never log sampled values; IO-38405 is not the uniqueness gate |
| 7 | GraphQL no-template (7.42%) already outperforms generic HTTP (84.37%) |
| 8 | 20.14% CF2-with-template blank rate verified; not a separate CF2 evaluator |
| 10 | 256-char middle truncation already shipped (IO-153375); no `{relativeURI}:row-N` fallbacks |

---

## 4.1 Start-Work Packs — everything an engineer needs to pick up each change

*Readiness legend: **Ready** = start today · **Needs input** = the listed decision/data must land first · **Blocked** = hard prerequisite.*

<details>
<summary><b>C1 · Expand assistant map</b> — Needs connector field research</summary>

| # | Step |
|---:|---|
| 1 | Prerequisite scope (corrected): D1 gates **only template-based profiles** — `templates` entries are dead config until the consumer reads them. **Field-only entries (`fields: [...]`) flow through `prioritizeFields` today and can ship independently of D1** |
| 2 | Add entries to map.js following the existing shape, e.g. `ebay: { fields: [...], templates: [] }` |
| 3 | Starter profiles to validate: eBay (orderId, itemId, lineItemId, sku), Walmart (purchaseOrderId, wpid, sku, itemId), Slack (id, ts, channel), Gainsight (Gsid, id). Validate against 5–10 real response samples per connector — do not ship unvalidated fields |
| 4 | Acceptance: map lookup returns the new profile; one unit test per connector shape |

**Tests:** `utils/tracekey/__tests__/patternFinder.test.js` · `utils/tracekey-common/__tests__/traceKeyUtil.test.js` · `flow-management/__tests__/unit/errorManagement/getTraceKeyPattern.test.js`

</details>

<details>
<summary><b>C2 · Detect Shopify/Amazon HTTP</b> — Needs sign-off on host detection table</summary>

| Host rule (anchored, on the **parsed** hostname) | Resolves to |
|---|---|
| `hostname.endsWith('.myshopify.com')` | shopify profile |
| `/^sellingpartnerapi(-[a-z]{2,4})?\.amazon\.com$/` (explicit regional variants) | new amazonsp profile |
| `/^mws(?:-[a-z]{2})?\.amazonservices\.[a-z.]+$/` (covers regional hosts like `mws-eu.amazonservices.com`) | amazonmws profile |

Parse `baseURI` with the URL API and match the extracted hostname **anchored** — never
substring "contains" matching, which deceptive or malformed URLs (path segments, userinfo
tricks) can spoof into a false connector classification.

| # | Step |
|---:|---|
| 1 | Trigger condition (corrected): run inference when there is **no explicit `connection.assistant`** and `connection.type` is generic (`http` / `rest`) — equivalently, after the map lookup at patternFinder.js:108–110 returns no profile. Note: `connection.type` is always set for generic HTTP, so a naive "assistant \|\| type is empty" guard would never fire. Never override a real assistant |
| 2 | Amazon SP-API: dedicated map profile after response validation (AmazonOrderId, orderId, ASIN, sku) — do not reuse the MWS profile |
| 3 | Acceptance: one test per host rule + negative test for arbitrary hosts |

</details>

<details>
<summary><b>C3 · Extract Handlebars from URI</b> — Ready</summary>

| # | Step |
|---:|---|
| 1 | Replace the last-segment prefix logic (patternFinder.js:294–297) with an all-segment extractor. **Prefer the Handlebars parser** (already a patternFinder dependency) over a regex: production URIs commonly use triple braces (`{{{data.id}}}.json`), and while a plain `\{\{…\}\}` regex does match triple braces by offset, it fails on bracketed paths (`record.items[0].id`) and cannot distinguish helpers. If a regex is used anyway, explicitly support double and triple braces |
| 2 | Take the leaf name of each `record.*` / `data.*` path; skip helpers and complex expressions |
| 3 | Candidate order: extracted fields first, then URI nouns, then dictionary |
| 4 | Acceptance: `/orders/{{record.orderId}}/items` yields orderId as top candidate; `{{{data.id}}}.json` yields id; `{{join a b}}` and `{{{encodeURI …}}}` ignored |

</details>

<details>
<summary><b>C4 · Widen identifier fallbacks</b> — Blocked on Change 6 gate</summary>

| # | Step |
|---:|---|
| 1 | HTTP-only ordered list: uuid, externalId, key, ref, code, number, sku — appended after explicit/URI candidates in patternFinder.js:274–311 |
| 1a | Case handling: runtime matching is **exact property-name** matching, so bare `uuid` misses `UUID`/`Uuid` and `sku` misses `SKU`. Generate case variants per name (the pattern the existing dictionary uses — 18 variants over 5 base names) or implement case-insensitive matching scoped to this HTTP list |
| 2 | Never touch the global dictionary in tracekey-common |
| 3 | Ship only after the C6 gate can reject non-unique candidates at runtime |

</details>

<details>
<summary><b>C5 · Response-batch inference</b> — Needs design review (contract below is the starting point)</summary>

**Proposed API contract**

```js
// utils/tracekey-common/lib/tracekey/batchInference.js
selectTraceKeyField(records, candidates, options)
// → { field, path, reason, distinctCount, rowCount }
// reason ∈ { UNIQUE_MATCH, NOT_UNIQUE, NO_CANDIDATE, EMPTY_PAGE }
// No separate `confidence` in v1 — distinctCount/rowCount carries the
// selection-quality signal; a derived score can be added later if needed.
```

| # | Step |
|---:|---|
| 1 | Deterministic: fixed candidate order, first candidate that passes wins |
| 2 | Job stickiness: cache the selected field per `_flowId + _exportId`; v1 uniqueness is page-scoped only |
| 3 | Integration: exportDataConverter.js:26–104 selects once per composite page (adaptor path); flow-execution `injectTraceKeys` (abstractBranchedFlowMessageProcessor.js:3213–3239) selects once per full worker page; endpoint-service preview mirrors via responseTransformUtil |
| 4 | Logging: field name, counts and reason only — never sampled record values |
| 5 | Edge cases to settle in design review: **minimum sample size** (a one-row page is trivially unique — require N ≥ ~10 or defer selection), empty pages, array/object/mixed-type values, composite candidates |
| 6 | First-page uncertainty and **job-cache invalidation** on schema drift mid-job; cross-page collisions are out of scope in v1 — document as a known limitation |
| 7 | Run in **shadow mode first**: compute and log the selection without stamping; enable stamping only after shadow metrics validate the selection quality |

</details>

<details>
<summary><b>C6 · Pre-stamp uniqueness gate</b> — Needs Product decision on user-template enforcement</summary>

| # | Step |
|---:|---|
| 1 | Gate v1: accept a candidate only if distinct non-null values === non-null rows AND non-null coverage ≥ 90% of the page (**90% is a proposed threshold, pending design review and shadow-mode data validation — see steps 5–6**); else fall through; if none pass, null + reason. Applies at both batch points: adaptor `exportDataConverter` and flow-execution `injectTraceKeys` |
| 2 | Inferred keys only. User templates never blocked at runtime — preview warning with collision count; template enforcement requires Product approval |
| 3 | Endpoint output: `{ traceKeysDuplicate, duplicateCount, source: template\|inferred, reason }` (responseTransformUtil.js:102–105, 129–141) |
| 4 | UI: CeligoTraceKeyWarning shows collision count and source |
| 5 | Edge cases to settle in design review: define the fate of the allowed ≤10% null rows (stamped null + reason vs excluded from the gate calculation), minimum sample size before the gate is trusted, and behavior on empty pages |
| 6 | **Shadow mode before enforcement**: log would-be rejections (candidate, collision count, reason) without changing stamping; flip to enforcement only after shadow data confirms the gate is not rejecting good keys |

</details>

<details>
<summary><b>C7 · GraphQL support</b> — Needs 2–3 real GraphQL export configs before coding</summary>

| # | Step |
|---:|---|
| 1 | Detect via connection formType `graph_ql` / `assistant_graphql` (connection.js:317–324) |
| 2 | Candidate order: from `export.http.response.resourcePath`, **strip known envelope segments** (`data`, `edges`, `node`, `nodes`, `items`, `results`) and select the semantic resource — a naive "leaf of resourcePath" would pick `edges` from `data.orders.edges`. Then id / node.id from the emitted record shape (Relay: `edges[].node.id`); query-body parsing last resort |
| 3 | Do not raise the global BFS depth; keep GraphQL-specific handling in patternFinder |

</details>

<details>
<summary><b>C8 · CF2 parity remainder</b> — Ship D2 first; blocked on ~10 CF2 production samples</summary>

| # | Step |
|---:|---|
| 1 | Pull ~10 CF2 exports joining configured traceKeyTemplate to one parsed record each; classify blanks into wrong-path vs eval-failure |
| 2 | Fix options after classification: trace-key-only `data` alias in main.js:201–228 (preferred) or nested-path guidance in UI. Never change baseHBDelegate globally — shared with webhook.successBody |
| 3 | Test matrix: `{{id}}`, `{{record.id}}`, `{{data.id}}`, `{{record.resource.id}}` against wrapped and unwrapped records, legacy and CF2 |

</details>

<details>
<summary><b>C9 · Reduce noisy URI segments</b> — Ready (final word list reviewed in PR)</summary>

| # | Step |
|---:|---|
| 1 | Lowercase each segment before the exclusion check (patternFinder.js:286–309) |
| 2 | Starter exclusion set (conservative): api, rest, graphql, search, export, list, bulk, batch, data. Prefer deprioritizing ambiguous words over hard-excluding |
| 3 | Acceptance: 'API' and 'Search' no longer candidates; orders/customers unaffected |

</details>

<details>
<summary><b>C10 · Explicit no-reliable-key state</b> — Needs sign-off on reason-code enum (proposal below)</summary>

**Proposed result API and reason enum**

```js
getTraceKeyResult(record, template, traceKeyPattern, _flowId)
// → { value, source: 'template'|'custom'|'pattern'|null,
//     field, reason, truncated }
// reason ∈ { OK, TEMPLATE_EVAL_FAILED, TEMPLATE_EMPTY, NO_CANDIDATE_FIELD,
//            CANDIDATES_NOT_UNIQUE, EMPTY_RECORD }
// truncated: boolean result flag — truncation still produces a valid key,
// so it is NOT a no-key reason
// v1 source stops at 'pattern': map-vs-dictionary provenance is lost today
// because prioritizeFields merges all candidates into one flat string array.
// Granular map|dictionary sourcing requires annotated candidates
// ({ field, source }) — a bigger contract change, deferred to v2.
```

| # | Step |
|---:|---|
| 1 | getTraceKey delegates to getTraceKeyResult; string/null contract unchanged |
| 2 | Propagate: exportDataConverter page metadata → qgmw.js:186–190 / injectTraceKeys → preview `dataRecordTraceKeyDetails[]` → UI all-missing state |
| 3 | Reasons are metadata only — never stamp a sentinel value as the key |

</details>

---

## 4.2 CF 2.0 Template Evaluation Divergence — Root Cause (Spike Deliverable 4)

**Question from the tracker:** why does CF 2.0 leave 20.1% of records blank when a trace-key
template is configured, while legacy HTTP with a template blanks only 0.24%?

**Conclusion:** there is **no separate CF 2.0 evaluation path**. CF2 and legacy HTTP stamp
trace keys through the exact same evaluator — `getTraceKeyValueByTemplate()` in
`tracekey-common/lib/tracekey/main.js` calling `hbUtil.evalExpressionSync(template, ctx, { strict: true })`.
The divergence is not *how* templates are evaluated but *what they are evaluated against*.
Findings below are labeled by confidence.

### Findings by confidence

| Confidence | Finding | Evidence |
|---|---|---|
| **Confirmed** | Shared evaluator — both cohorts funnel into the same `tracekey-common` functions; a separate CF2 evaluation path is disproven | Code reading of main.js and both adaptor call paths |
| **Confirmed** | Silent wrapper handling (D2) — `evalExpressionSync` returns `{ value }` or `{ error }`; `main.js:201–211` passes the wrapper into `processTraceKey()`, so every strict-mode failure becomes a null key with no log and no reason | main.js:201–211; no template-failure telemetry exists today |
| **Strong hypothesis** | Wrong record path/context — CF2 resource-wrapped records (id at `record.addon.id`, not `record.id`) plus the missing `data.*` alias in bare `BaseHbDelegate` (legacy `oldContext.js:45–53` exposes both `data.*` and root fields) cause strict failures for templates written against the legacy shape | CF2 response fixtures in http-adaptor; context comparison — **not yet joined to production samples** |
| **Unverified** | Production cause distribution — how much of the 20.14% is wrong-path vs eval-failure vs something else | Requires D2 telemetry + ~10 CF2 samples (C8 pack) |
| **Unverified** | Duplicate mechanism (9.50% vs 1.50% legacy) — partially-resolving templates producing repeated static-ish values is plausible but unproven | Requires the same sample joins |

### Adjacent defect — D4 is not a cause of this cohort

Preview omitting `traceKeyPattern` (D4) does **not** explain the with-template blank rate:
an explicit template returns from evaluation at `main.js:41–43` *before* the pattern fallback
is ever consulted. D4 breaks preview parity only for the **no-template** path; it ships in
Phase 1 for that reason, not as a CF2-divergence fix.

### What was ruled out

- **A separate CF2 evaluator** — disproven by code reading; both cohorts funnel into the same
  `tracekey-common` functions.
- **The D1 `templates`/`template` contract defect** — does not explain this cohort. Explicit
  user templates return *before* the map fallback, so the map contract never runs here.
- **The D3 CF2 assistant-map bypass** — real, but it affects the **no-template** cohort
  (77.34% blank), not the 20.14% with-template cohort.

### Quantification — why the split isn't in this spike

Splitting the 20.14% into "wrong path" vs "evaluation failure" is impossible from today's logs
because the silent wrapper handling swallows the distinction: both cases land as the same null.
The split becomes measurable once D2 ships (Phase 1) and error codes appear in telemetry. The C8
start-work pack then joins ~10 CF2 exports' configured template strings to one parsed record
sample each to classify the remainder.

**Follow-up required:** file a dedicated data spike ("Classify CF2 with-template blanks using
D2 telemetry + ~10 template-to-record-shape sample joins"). Until it completes, deliverable 4
status is: **mechanisms confirmed; production cause distribution pending follow-up**. Because
the tracker explicitly asks for root cause, either complete the sampling before closing
IO-208154 or keep the spike open with the linked follow-up **explicitly accepted by
stakeholders** as the completion path.

### Fix mapping

| Fix | Phase | What it addresses |
|---|---|---|
| D2 — unwrap `{value, error}`, aggregate error telemetry, return reason | 1 | Makes silent failures visible; enables the wrong-path vs eval-failure split |
| D4 — carry `traceKeyPattern` into preview (see D4 scope in Section 3) | 1 | Restores preview parity for the **no-template** path — adjacent defect, not a cause of this cohort |
| Change 8 remainder — trace-key-only `data` alias in main.js:201–228 (preferred) or nested-path guidance in UI | 3 | Addresses the wrong-path hypothesis and the `data.*` context gap; approach chosen after D2 telemetry classifies the blanks |

---

## 4.3 Root-Cause Coverage Matrix (Spike Deliverable 1)

The source Confluence page identifies nine root causes for HTTP/REST trace-key failures.
This matrix records the audit verdict for each, making the validation auditable.

| RC | Confluence root cause | Verdict | Evidence | Changes | Remaining validation |
|---|---|---|---|---|---|
| 1 | Assistant map covers only 22 of 49+ HTTP apps (eBay 0%, Walmart 3%, Slack 8.9% stamped) | **Confirmed** | map.js has 22 entries; the named connectors are absent | C1, C2 (+ D1, D3 prerequisites) | Field research per connector before merge |
| 2 | URI-only heuristic, no response inspection — 89% of missing generic HTTP keys from ~2,259 custom API exports | **Confirmed** | patternFinder derives candidates solely from `http.relativeURI` | C4, C5 | Traffic-share modeling to size the impact |
| 3 | Templates make duplicates worse for generic HTTP (3.9% → 21.5% job-level) | **Confirmed (data)** — mechanism is the absence of preview-time uniqueness validation | Evidence workbook; no gate exists anywhere in the code | C6 | Product decision on user-template enforcement |
| 4 | CF 2.0 underperforms legacy HTTP with templates (20.1% blank, 9.5% dup) | **Modified** — "different evaluation path" is disproven; wrong record shape + silent strict failures instead (see 4.2) | Section 4.2 findings-by-confidence table | C8, D2 (D4 adjacent) | ~10 production samples after D2 telemetry |
| 5 | GraphQL endpoints are a dead end (`*graphql` candidate matches nothing) | **Confirmed** | URI inference emits `*graphql`; runtime regex `/.*(graphql).?id$/i` matches nothing meaningful | C7 | 2–3 real GraphQL export configs |
| 6 | Handlebars field reference in URI is discarded | **Confirmed** | patternFinder.js:294–297 keeps only the literal prefix before `{{` | C3 | None — ready to build |
| 7 | `api` is the only hardcoded URI exclusion | **Confirmed with modification** — `v1`/`v2` are already rejected by the alpha-only regex; `API` survives because the check is case-sensitive | patternFinder.js:286–309 | C9 | Final word-list review in PR |
| 8 | No uniqueness check anywhere | **Confirmed** | `findTraceKey` stamps the first matching field, no distinctness validation | C5, C6 | Gate edge-case design (see C5/C6 packs) |
| 9 | REST-to-HTTP conversion loses resource metadata | **Not verified in this spike** | REST routes to the same inference function, but the conversion path itself was not audited | — | **File a dedicated follow-up audit ticket** for the conversion path (what metadata existed pre-conversion, what survives on the HTTP doc) |

---

## 5. Release Plan — Phases in Shipping Order

| Step | Phase | Risk | Ship | Notes |
|---:|---|---|---|---|
| 1 | 1 · Correctness | Low | D1 consumer + D2 + D5 (`tracekey-common/main.js`) | Publish first |
| 2 | 1 · Correctness | Low | D1 producer + Change 3 + Change 9 (`tracekey/patternFinder.js`) | After step 1 |
| 3 | 1 · Correctness | Low | D4 (`endpoint-service/responseTransformUtil.js`) | After step 1 published |
| 4 | 2 · Connector coverage | Medium | D3 — CF2 legacyId resolution | em-util/flow-management getTraceKeyPattern |
| 5 | 2 · Connector coverage | Medium | Change 1 + Change 2 | Recommended train with D3 (not a hard dependency — see Dependency Rules) |
| 6 | 3 · Parity + GraphQL | Medium | Change 8 remainder | After D2 error telemetry proves cause |
| 7 | 3 · Parity + GraphQL | Medium | Change 7 — GraphQL resourcePath | Parallel with step 6 |
| 8 | 4 · Batch framework | High | Change 5 + Change 6 | One project |
| 9 | 4 · Batch framework | High | Change 10 — reason metadata | On the 5/6 result API |
| 10 | 4 · Batch framework | High | Change 4 — widened fallbacks | Only after step 8 gate exists |

### Dependency Rules

| Rule | Items |
|---|---|
| Ship independently | D5, Change 9, Change 3, D2 |
| Hard ordering | D1 consumer before D1 producer |
| Hard batches | Change 5 + Change 6 (one framework) · Change 4 only after the Change 6 gate exists · Change 10 builds on the 5/6 result API |
| Recommended (not hard dependencies) | D3 + Change 1 + Change 2 as one train — Change 1 alone already improves connections that have a legacy assistant; D3/C2 extend the reach to CF2 and generic HTTP · D4 after D1 consumer — D4 alone still fixes the fields-based preview fallback; only the map-template portion stays dead until D1 |
| Conditional | Change 7 fallback waits for Change 5; Change 8 waits for D2 telemetry |
| Hard constraint (D6) | Every train publishes tracekey-common → tracekey → bumps identical pins across all consumers |

### Phase Summary

| Phase | Risk | Defects | Changes | Goal |
|---|---|---|---|---|
| 1 · Correctness | Low | D1, D2, D4, D5 | 3, 9 | Fix bugs; improve URI inference |
| 2 · Connector coverage | Medium | D3 | 1, 2 | Map expansion; CF2 + known HTTP profiles |
| 3 · Parity + GraphQL | Medium | — | 7, 8 | Close CF2 template gap; GraphQL resourcePath |
| 4 · Batch framework | High | — | 5, 6, 10, then 4 | Inference, uniqueness gate, reason codes |

---

## 6. Expected Metric Impact by Phase (Hypotheses)

*These are **hypotheses, not modeled commitments** — exact gains depend on connector traffic mix, which has not been measured per cohort. Before each phase deploys, define a measurable acceptance threshold (e.g. "no-key % in the target cohort improves by ≥ X points within 4 weeks") and a rollback criterion (e.g. "duplicate % must not rise by more than Y points"). Verify with re-run workbook queries plus the D2 template-failure metric, using denominators that exclude the 529 health-check exports (see Section 2 provenance).*

| Phase | Coverage (hypothesis) | Duplicates (hypothesis) | Measurement / Explainability |
|---|---|---|---|
| After Phase 1 | Small direct gain off 84.37% legacy no-template | Fewer wrong keys from URI noise (Change 9) | D2 makes template-failure rate measurable; D4 restores no-template preview parity |
| After Phase 2 | Largest no-template lever — targeted connectors improve materially; gain is proportional to their traffic share, which must be measured before setting a numeric target | Roughly flat — curated map fields, low collision risk | — |
| After Phase 3 | CF2 with-template no-key (20.14%) reduced materially — legacy-template 0.24% is the theoretical ceiling; GraphQL no-template (7.42%) reduced — model a target after D2 telemetry lands | CF2 template duplicates (9.50%): partial improvement via nested-path guidance | — |
| After Phase 4 | Batch inference catches long-tail APIs with non-standard ID fields | Uniqueness gate reduces inferred-key duplicates (0.33%–10.79% today); validate in shadow mode before claiming a number | Change 10 reason codes eliminate the "unknown missing" bucket |

---

## 7. Phase 1 Delivery — Three PRs, One Release Train

| PR | Repo · File | Contains | Order |
|---:|---|---|---|
| 1 | `integrator-common-util` · `tracekey-common/lib/tracekey/main.js` | D1 consumer · D2 · D5 | Publish first |
| 2 | `integrator-common-util` · `tracekey/lib/tracekey/patternFinder.js` | D1 producer · Change 3 · Change 9 (new feature code — not automatic from defect fixes) | After PR 1 |
| 3 | `endpoint-service` · `responseTransformUtil.js` + preview API contract (pattern must originate upstream — see D4 scope in Section 3) | D4 | After PR 1 published |

---

## 8. Deployment Chain (Every Phase)

| Step | Action | Scope |
|---:|---|---|
| 1 | Publish tracekey-common; bump @celigo/tracekey | @celigo/tracekey-common |
| 2 | Publish with new tracekey-common dependency | @celigo/tracekey |
| 3 | Bump tracekey in em-util; deploy flow-management | em-util → flow-management |
| 4 | Release adaptor tag + flow-execution with identical tracekey versions | integrator-adaptor + flow-execution |
| 5 | Bump worker pins; rolling deploy | integrator-workers |
| 6 | Deploy preview sync | endpoint-service + integrator-ui |

---

## 9. Verified Sources

| Source | Items |
|---|---|
| Jira | IO-208154 (spike) · IO-38405 (null over generated keys) · IO-153375 (256-char cap, shipped) |
| Confluence | 2290384905 (HTTP adaptor recommendations) · 2167341069 (broader trace-key actions) |
| Evidence workbook | Google Sheet "Summary w/ config from logs" |
| Main revisions | integrator-common-util 078fe54 · integrator-adaptor 1a8be58 · http-adaptor c0318a7 · flow-execution 347705f · endpoint-service 00dc15a · em-util 1302bda · integrator-models 4743a6b · flow-management 670fb54 · integrator-workers c7839b9 · integrator-ui c287211 |
