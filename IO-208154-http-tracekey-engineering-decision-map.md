# IO-208154 · HTTP Trace-Key Engineering Decision Map

Fresh validation of all ten proposals against clean current-main clones, the linked Jira/Confluence material, and the evidence workbook.

---

## 1. Executive Summary

**Bottom line:** Ship the small correctness fixes first, but do not implement the ten items as ten isolated PRs. Changes 5 and 6 require one batch-aware framework; Change 8 is three concrete parity defects; and Change 10 should expose reasons without generating a synthetic key. The largest operational risk is package skew across design-time, workers, and preview.

| Metric | Value |
|---|---:|
| Changes mapped to code | 10 / 10 |
| Current defects found (D1–D6) | 6 |
| Repositories in rollout path | 8 |
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

---

## 3. Defects Found During the Audit (D1–D6)

D1–D6 are **not** part of the ten proposed changes. They are existing bugs discovered while verifying those proposals. Each needs its own Jira ticket.

| Defect | Location & issue | Fix summary | What the fix solves |
|---|---|---|---|
| **D1** | Producer returns `templates` (patternFinder.js:113/143); consumer reads `template` (main.js:47–52) | Consumer accepts both keys; producer emits `template` (keep `templates` one release) | NetSuite (`{{Images.image_record_id}}`) and Zendesk (`{{name}}-{{email}}`) map templates start working; unblocks Change 1 |
| **D2** | evalExpressionSync wrapper passed to processTraceKey (main.js:201–211); failures collapse to null | Unwrap wrapper; rate-limited warn with error code only; return null on error | Separates "template failed" from "field empty"; foundation for Change 10 and Change 8 attribution |
| **D3** | Map lookup uses `assistant \|\| type`; CF2 connections often have no assistant (connection.js:5507–5517) | Resolve legacyId at pattern-build time when `_httpConnectorId` present | CF2 connectors inherit curated map profiles; attacks 77.34% CF2 no-template cohort |
| **D4** | Preview calls getTraceKey without traceKeyPattern (responseTransformUtil.js:11) | Thread traceKeyPattern through preview fallback path | Preview matches production trace-key resolution and duplicate warnings |
| **D5** | flowCache is unbounded array (main.js:21, 56–59); undercounts missing-key telemetry | Replace with size-capped Set; keep once-per-flow logging | Bounds worker memory; does **not** cause duplicates — only undercounts missing-key logs |
| **D6** | Consumer version skew (8.0.1–8.0.19 across repos) | Publish tracekey-common → tracekey; bump identical pins every phase | Prevents design-time, preview, and production running different algorithms |

---

## 4. The Ten Proposed Changes

| # | Title | Status | Size | Phase | Primary owner | Primary locations |
|---:|---|---|---|---:|---|---|
| 1 | Expand assistant map | Proceed after D1 | S | 2 | integrator-common-util | map.js, patternFinder.js, main.js |
| 2 | Detect Shopify/Amazon HTTP | Rework | S | 2 | integrator-common-util | patternFinder.js, connection.js, map.js |
| 3 | Extract Handlebars from URI | Proceed | S | 1 | integrator-common-util | patternFinder.js:274–311 |
| 4 | Widen identifier fallbacks | HTTP-scope | S | 4 | integrator-common-util | patternFinder.js:274–311 |
| 5 | Response-batch inference | Framework | M/L | 4 | common-util + adaptor + endpoint/UI only | tracekey-common, exportDataConverter, abstractExport, endpoint |
| 6 | Pre-stamp uniqueness gate | With Change 5 | M | 4 | common-util + adaptor + endpoint/UI only | tracekey-common, adaptor, endpoint, UI |
| 7 | GraphQL support | Proceed | S | 3 | integrator-common-util | connection.js, patternFinder.js, main.js |
| 8 | CF2/template divergence | Needs samples | S | 3 | common-util + endpoint | main.js, baseHBDelegate.js, endpoint |
| 9 | Reduce noisy URI segments | Proceed | S | 1 | integrator-common-util | patternFinder.js:286–309 |
| 10 | Explicit no-reliable-key state | Proceed | S | 4 | common-util + adaptor + FE + endpoint/UI | main.js, abstractAdaptor, qgmw, endpoint, UI |

*Size legend: size = total effort (research, tests, rollout) — not diff size. Most code diffs are tens of lines; Changes 5 and 6 are the only large builds (breadth across four repos).*

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
| 1 | Prerequisite: D1 fix merged first (new `templates` entries are dead config until then) |
| 2 | Add entries to map.js following the existing shape, e.g. `ebay: { fields: [...], templates: [] }` |
| 3 | Starter profiles to validate: eBay (orderId, itemId, lineItemId, sku), Walmart (purchaseOrderId, wpid, sku, itemId), Slack (id, ts, channel), Gainsight (Gsid, id). Validate against 5–10 real response samples per connector — do not ship unvalidated fields |
| 4 | Acceptance: map lookup returns the new profile; one unit test per connector shape |

**Tests:** `utils/tracekey/__tests__/patternFinder.test.js` · `utils/tracekey-common/__tests__/traceKeyUtil.test.js` · `flow-management/__tests__/unit/errorManagement/getTraceKeyPattern.test.js`

</details>

<details>
<summary><b>C2 · Detect Shopify/Amazon HTTP</b> — Needs sign-off on host detection table</summary>

| Host rule | Resolves to |
|---|---|
| host contains `.myshopify.com` | shopify profile |
| host matches `sellingpartnerapi-*.amazon.com` | new amazonsp profile |
| host matches `mws.amazonservices.*` | amazonmws profile |

| # | Step |
|---:|---|
| 1 | Derive `inferredAssistant` in patternFinder.js:29–82 only when `connection.assistant \|\| connection.type` is empty; pass into map prioritization at 108–143; never override a real assistant |
| 2 | Amazon SP-API: dedicated map profile after response validation (AmazonOrderId, orderId, ASIN, sku) — do not reuse the MWS profile |
| 3 | Acceptance: one test per host rule + negative test for arbitrary hosts |

</details>

<details>
<summary><b>C3 · Extract Handlebars from URI</b> — Ready</summary>

| # | Step |
|---:|---|
| 1 | Replace the last-segment prefix logic (patternFinder.js:294–297) with an all-segment extractor: `/\{\{\s*(?:record|data)\.([A-Za-z0-9_.]+)\s*\}\}/g`; take the leaf name; skip helpers/complex expressions |
| 2 | Candidate order: extracted fields first, then URI nouns, then dictionary |
| 3 | Acceptance: `/orders/{{record.orderId}}/items` yields orderId as top candidate; `{{join a b}}` ignored |

</details>

<details>
<summary><b>C4 · Widen identifier fallbacks</b> — Blocked on Change 6 gate</summary>

| # | Step |
|---:|---|
| 1 | HTTP-only ordered list: uuid, externalId, key, ref, code, number, sku — appended after explicit/URI candidates in patternFinder.js:274–311 |
| 2 | Never touch the global dictionary in tracekey-common |
| 3 | Ship only after the C6 gate can reject non-unique candidates at runtime |

</details>

<details>
<summary><b>C5 · Response-batch inference</b> — Needs design review (contract below is the starting point)</summary>

**Proposed API contract**

```js
// utils/tracekey-common/lib/tracekey/batchInference.js
selectTraceKeyField(records, candidates, options)
// → { field, path, confidence, reason, distinctCount, rowCount }
// reason ∈ { UNIQUE_MATCH, NOT_UNIQUE, NO_CANDIDATE, EMPTY_PAGE }
```

| # | Step |
|---:|---|
| 1 | Deterministic: fixed candidate order, first candidate that passes wins |
| 2 | Job stickiness: cache the selected field per `_flowId + _exportId`; v1 uniqueness is page-scoped only |
| 3 | Integration: exportDataConverter.js:26–104 selects once per composite page; endpoint-service preview mirrors via responseTransformUtil |
| 4 | Logging: field name, counts and reason only — never sampled record values |

</details>

<details>
<summary><b>C6 · Pre-stamp uniqueness gate</b> — Needs Product decision on user-template enforcement</summary>

| # | Step |
|---:|---|
| 1 | Gate v1: accept a candidate only if distinct non-null values === non-null rows AND non-null coverage ≥ 90% of the page; else fall through; if none pass, null + reason |
| 2 | Inferred keys only. User templates never blocked at runtime — preview warning with collision count; template enforcement requires Product approval |
| 3 | Endpoint output: `{ traceKeysDuplicate, duplicateCount, source: template\|inferred, reason }` (responseTransformUtil.js:102–105, 129–141) |
| 4 | UI: CeligoTraceKeyWarning shows collision count and source |

</details>

<details>
<summary><b>C7 · GraphQL support</b> — Needs 2–3 real GraphQL export configs before coding</summary>

| # | Step |
|---:|---|
| 1 | Detect via connection formType `graph_ql` / `assistant_graphql` (connection.js:317–324) |
| 2 | Candidate order: leaf of `export.http.response.resourcePath` first, then id / node.id from emitted record shape (Relay: `edges[].node.id`); query-body parsing last resort |
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
// → { value, source: 'template'|'custom'|'map'|'dictionary'|null, field, reason }
// reason ∈ { OK, TEMPLATE_EVAL_FAILED, TEMPLATE_EMPTY, NO_CANDIDATE_FIELD,
//            CANDIDATES_NOT_UNIQUE, EMPTY_RECORD, TRUNCATED }
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
The divergence is not *how* templates are evaluated but *what they are evaluated against*,
compounded by two defects that hide the failures.

### The three mechanisms

| # | Mechanism | What happens | Evidence |
|---|---|---|---|
| A | **Resource-wrapped records** | CF2 connectors emit records wrapped in a resource envelope (e.g. the record is `{ addon: {...} }`, so the id lives at `record.addon.id`). Users write `{{record.id}}` assuming the legacy shape, where the record *is* the resource. Under `strict: true`, the missing path fails the evaluation | CF2 response fixtures in http-adaptor; template strings joined to parsed record shapes |
| B | **Strict failures collapse silently (D2)** | `evalExpressionSync` returns `{ value }` or `{ error }`. `main.js:201–211` passes the wrapper straight into `processTraceKey()`, so a strict-mode error becomes a null key with no log and no reason. Every wrong-path template from mechanism A turns into an invisible blank | main.js:201–211; no error telemetry exists for template failures today |
| C | **Preview omits the pattern (D4)** | endpoint-service preview calls `getTraceKey(record, template)` without `traceKeyPattern` (responseTransformUtil.js:11), so design-time preview can disagree with production. Users have no signal at configure time that their template resolves to nothing in the real worker path | responseTransformUtil.js:11 vs worker path in flow-execution |

A secondary context gap: legacy HTTP's Handlebars context (`oldContext.js:45–53`) exposes both
`data.*` and root-level fields, while the bare `BaseHbDelegate` used in trace-key evaluation
does not provide the `data.*` alias. Templates written as `{{data.id}}` — valid elsewhere in
legacy HTTP — fail silently under trace-key evaluation for the same D2 reason.

### What was ruled out

- **A separate CF2 evaluator** — disproven by code reading; both cohorts funnel into the same
  `tracekey-common` functions.
- **The D1 `templates`/`template` contract defect** — does not explain this cohort. Explicit
  user templates return *before* the map fallback, so the map contract never runs here.
- **The D3 CF2 assistant-map bypass** — real, but it affects the **no-template** cohort
  (77.34% blank), not the 20.14% with-template cohort.

### Why the duplicate rate is also high (9.50% vs 1.50% legacy)

The same wrong-shape templates that usually fail can partially resolve — e.g. a template with
static text plus a field that is missing on most records evaluates to the same static-ish value
repeatedly, producing collisions instead of blanks. Blank rate and duplicate rate are two
symptoms of one cause: templates written against the wrong record shape.

### Quantification — why the split isn't in this spike

Splitting the 20.14% into "wrong path" vs "evaluation failure" is impossible from today's logs
because mechanism B swallows the distinction: both cases land as the same null. The split
becomes measurable once D2 ships (Phase 1) and error codes appear in telemetry. The C8
start-work pack then joins ~10 CF2 exports' configured template strings to one parsed record
sample each to classify the remainder.

### Fix mapping

| Fix | Phase | What it addresses |
|---|---|---|
| D2 — unwrap `{value, error}`, log error code, return reason | 1 | Makes mechanism B visible; enables the wrong-path vs eval-failure split |
| D4 — thread `traceKeyPattern` through preview | 1 | Makes mechanism C honest; preview matches production |
| Change 8 remainder — trace-key-only `data` alias in main.js:201–228 (preferred) or nested-path guidance in UI | 3 | Addresses mechanism A and the `data.*` context gap; approach chosen after D2 telemetry classifies the blanks |

---

## 5. Release Plan — Phases in Shipping Order

| Step | Phase | Risk | Ship | Notes |
|---:|---|---|---|---|
| 1 | 1 · Correctness | Low | D1 consumer + D2 + D5 (`tracekey-common/main.js`) | Publish first |
| 2 | 1 · Correctness | Low | D1 producer + Change 3 + Change 9 (`tracekey/patternFinder.js`) | After step 1 |
| 3 | 1 · Correctness | Low | D4 (`endpoint-service/responseTransformUtil.js`) | After step 1 published |
| 4 | 2 · Connector coverage | Medium | D3 — CF2 legacyId resolution | em-util/flow-management getTraceKeyPattern |
| 5 | 2 · Connector coverage | Medium | Change 1 + Change 2 | Batched with D3 |
| 6 | 3 · Parity + GraphQL | Medium | Change 8 remainder | After D2 error telemetry proves cause |
| 7 | 3 · Parity + GraphQL | Medium | Change 7 — GraphQL resourcePath | Parallel with step 6 |
| 8 | 4 · Batch framework | High | Change 5 + Change 6 | One project |
| 9 | 4 · Batch framework | High | Change 10 — reason metadata | On the 5/6 result API |
| 10 | 4 · Batch framework | High | Change 4 — widened fallbacks | Only after step 8 gate exists |

### Dependency Rules

| Rule | Items |
|---|---|
| Ship independently | D5, Change 9, Change 3, D2 |
| Ordered pairs | D1 consumer before D1 producer; D4 after D1 consumer |
| Must batch | D3 + Change 1 + Change 2 · Change 5 + Change 6 · Change 4 + Change 6 · Change 10 follows 5/6 |
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

## 6. Expected Metric Impact by Phase

*Directional targets only — exact gains depend on connector traffic mix. Verify each phase: re-run workbook queries (no-key % and duplicate % per cohort) plus D2 template-failure metric, four weeks pre/post deploy.*

| Phase | Coverage | Duplicates | Measurement / Explainability |
|---|---|---|---|
| After Phase 1 | Small direct gain (single-digit points off 84.37% legacy no-template) | Fewer wrong keys from URI noise (Change 9) | D2 makes template-failure rate measurable; D4 makes preview match production |
| After Phase 2 | Biggest no-template lever — targeted connectors drop from 84.37% / 77.34% toward near-complete keys | Roughly flat — curated map fields, low collision risk | — |
| After Phase 3 | CF2 with-template no-key: 20.14% → low single digits (legacy 0.24% is the ceiling); GraphQL no-template: 7.42% → under ~3% | CF2 template duplicates (9.50%): partial improvement via nested-path guidance | — |
| After Phase 4 | Batch inference catches long-tail APIs with non-standard ID fields | Uniqueness gate drives inferred-key duplicates (0.33%–10.79%) toward zero | Change 10 reason codes eliminate "unknown missing" bucket |

---

## 7. Phase 1 Delivery — Three PRs, One Release Train

| PR | Repo · File | Contains | Order |
|---:|---|---|---|
| 1 | `integrator-common-util` · `tracekey-common/lib/tracekey/main.js` | D1 consumer · D2 · D5 | Publish first |
| 2 | `integrator-common-util` · `tracekey/lib/tracekey/patternFinder.js` | D1 producer · Change 3 · Change 9 (new feature code — not automatic from defect fixes) | After PR 1 |
| 3 | `endpoint-service` · `responseTransformUtil.js` | D4 | After PR 1 published |

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
