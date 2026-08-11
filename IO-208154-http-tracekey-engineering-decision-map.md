# IO-208154 · HTTP Trace-Key Engineering Decision Map

Fresh validation of all ten proposals against clean current-main clones, the linked Jira/Confluence material, and the evidence workbook.

---

## 1. Executive Summary

**Bottom line:** Ship the small correctness fixes first, but do not implement the ten items as ten isolated PRs. Changes 5 and 6 require one batch-aware framework; Change 8 is three concrete parity defects; and Change 10 should expose reasons without generating a synthetic key. The largest operational risk is package skew across design-time, workers, and preview.

**Key stats**
- 10/10 changes mapped to code
- 6 current defects found (D1–D6)
- 8 repositories in rollout path
- 0 product source files changed (read-only audit)

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

| Defect | Location & issue | Fix summary |
|---|---|---|
| **D1** | Producer returns `templates` (patternFinder.js:113/143); consumer reads `template` (main.js:47–52) | Consumer accepts both keys; producer emits `template` (keep `templates` one release) |
| **D2** | evalExpressionSync wrapper passed to processTraceKey (main.js:201–211); failures collapse to null | Unwrap wrapper; rate-limited warn with error code only; return null on error |
| **D3** | Map lookup uses `assistant \|\| type`; CF2 connections often have no assistant (connection.js:5507–5517) | Resolve legacyId at pattern-build time when `_httpConnectorId` present |
| **D4** | Preview calls getTraceKey without traceKeyPattern (responseTransformUtil.js:11) | Thread traceKeyPattern through preview fallback path |
| **D5** | flowCache is unbounded array (main.js:21, 56–59); undercounts missing-key telemetry | Replace with size-capped Set; keep once-per-flow logging |
| **D6** | Consumer version skew (8.0.1–8.0.19 across repos) | Publish tracekey-common → tracekey; bump identical pins every phase |

**What each defect fix solves**
- **D1** — NetSuite (`{{Images.image_record_id}}`) and Zendesk (`{{name}}-{{email}}`) map templates start working; unblocks Change 1.
- **D2** — Separates "template failed" from "field empty"; foundation for Change 10 and Change 8 attribution.
- **D3** — CF2 connectors inherit curated map profiles; attacks 77.34% CF2 no-template missing-key cohort.
- **D4** — Preview matches production trace-key resolution and duplicate warnings.
- **D5** — Bounds worker memory; does **not** cause duplicates — only undercounts missing-key logs.
- **D6** — Prevents design-time, preview, and production running different algorithms.

---

## 4. The Ten Proposed Changes

| # | Title | Status | Size | Primary owner | Primary locations | Notes |
|---:|---|---|---|---|---|---|
| 1 | Expand assistant map | Proceed after D1 | S | integrator-common-util | map.js, patternFinder.js, main.js | Needs D3 + Change 2 for CF2/generic HTTP |
| 2 | Detect Shopify/Amazon HTTP | Rework | S | integrator-common-util | patternFinder.js, connection.js, map.js | Do not reuse Amazon MWS profile |
| 3 | Extract Handlebars from URI | Proceed | S | integrator-common-util | patternFinder.js:274–311 | Emit exact fields without `*` |
| 4 | Widen identifier fallbacks | HTTP-scope | S | integrator-common-util | patternFinder.js:274–311 | Only safe after Change 6 gate (Phase 4) |
| 5 | Response-batch inference | Framework | M/L | common-util + adaptor + endpoint/UI only | tracekey-common, exportDataConverter, abstractExport, endpoint | Page-scoped; cache field per job |
| 6 | Pre-stamp uniqueness gate | With Change 5 | M | common-util + adaptor + endpoint/UI only | tracekey-common, adaptor, endpoint, UI | IO-38405 is not this gate |
| 7 | GraphQL support | Proceed | S | integrator-common-util | connection.js, patternFinder.js, main.js | Prefer resourcePath / emitted shape |
| 8 | CF2/template divergence | Needs samples | S | common-util + endpoint | main.js, baseHBDelegate.js, endpoint | Same evaluator; resource-wrapped records + D2 failures |
| 9 | Reduce noisy URI segments | Proceed | S | integrator-common-util | patternFinder.js:286–309 | Normalized exclusions; v1/v2 already rejected |
| 10 | Explicit no-reliable-key state | Proceed | S | common-util + adaptor + FE + endpoint/UI | main.js, abstractAdaptor, qgmw, endpoint, UI | No `{relativeURI}:row-N` fallbacks |

*Size legend: size = total effort (research, tests, rollout) — not diff size. Most code diffs are tens of lines; Changes 5 and 6 are the only large builds (breadth across four repos).*

**Key caveats**
- Change 1: map entry runs only when `connection.assistant || connection.type` matches.
- Change 4: "four-field dictionary" is actually 18 case variants over 5 base names on current main.
- Changes 5/6: page-scoped and deterministic; never log sampled values.
- Change 7: GraphQL no-template (7.42%) already outperforms generic HTTP (84.37%).
- Change 8: 20.14% CF2-with-template blank rate verified; not a separate CF2 evaluator.
- Change 10: 256-char middle truncation already shipped (IO-153375).

### 4.5 Start-Work Packs — everything an engineer needs to pick up each change

*Readiness legend: Ready = start today · Needs input = the listed decision/data must land first · Blocked = hard prerequisite.*

**C1 · Expand assistant map — Needs connector field research**
- Prerequisite: D1 fix merged first (new `templates` entries are dead config until then).
- Add entries to map.js following the existing shape, e.g. `ebay: { fields: [...], templates: [] }`.
- Starter profiles to validate: eBay (orderId, itemId, lineItemId, sku), Walmart (purchaseOrderId, wpid, sku, itemId), Slack (id, ts, channel), Gainsight (Gsid, id). Validate against 5–10 real response samples per connector — do not ship unvalidated fields.
- Acceptance: map lookup returns the new profile; one unit test per connector shape.

**C2 · Detect Shopify/Amazon HTTP — Needs sign-off on host detection table**
- Detection table (starter): host contains `.myshopify.com` → shopify; `sellingpartnerapi-*.amazon.com` → new amazonsp profile; `mws.amazonservices.*` → amazonmws.
- Derive `inferredAssistant` in patternFinder.js:29–82 only when `connection.assistant || connection.type` is empty; never override a real assistant.
- Amazon SP-API: dedicated map profile after response validation (AmazonOrderId, orderId, ASIN, sku) — do not reuse the MWS profile.
- Acceptance: one test per host rule + negative test for arbitrary hosts.

**C3 · Extract Handlebars from URI — Ready**
- Replace the last-segment prefix logic (patternFinder.js:294–297) with an all-segment extractor: `/\{\{\s*(?:record|data)\.([A-Za-z0-9_.]+)\s*\}\}/g`; take the leaf name; skip helpers/complex expressions.
- Candidate order: extracted fields first, then URI nouns, then dictionary.
- Acceptance: `/orders/{{record.orderId}}/items` yields orderId as top candidate; `{{join a b}}` ignored.

**C4 · Widen identifier fallbacks — Blocked on Change 6 gate**
- HTTP-only ordered list: uuid, externalId, key, ref, code, number, sku — appended after explicit/URI candidates in patternFinder.js:274–311. Never touch the global dictionary in tracekey-common.
- Ship only after the C6 gate can reject non-unique candidates at runtime.

**C5 · Response-batch inference — Needs design review (contract below is the starting point)**
- New module `tracekey-common/lib/tracekey/batchInference.js`: `selectTraceKeyField(records, candidates, options)` → `{ field, path, confidence, reason, distinctCount, rowCount }`.
- Reason values: UNIQUE_MATCH, NOT_UNIQUE, NO_CANDIDATE, EMPTY_PAGE. Deterministic: fixed candidate order, first candidate that passes wins.
- Job stickiness: cache the selected field per `_flowId + _exportId`; v1 uniqueness is page-scoped only.
- Integration: exportDataConverter.js:26–104 selects once per composite page; endpoint-service preview mirrors via responseTransformUtil.
- Logging: field name, counts and reason only — never sampled record values.

**C6 · Pre-stamp uniqueness gate — Needs Product decision on user-template enforcement**
- Gate v1: accept a candidate only if distinct non-null values === non-null rows AND non-null coverage ≥ 90% of the page; else fall through; if none pass, null + reason.
- Inferred keys only. User templates never blocked at runtime — preview warning with collision count; template enforcement requires Product approval.
- Endpoint output: `{ traceKeysDuplicate, duplicateCount, source: template|inferred, reason }` (responseTransformUtil.js:102–105, 129–141).
- UI: CeligoTraceKeyWarning shows collision count and source.

**C7 · GraphQL support — Needs 2–3 real GraphQL export configs before coding**
- Detect via connection formType `graph_ql` / `assistant_graphql` (connection.js:317–324).
- Candidate order: leaf of `export.http.response.resourcePath` first, then id / node.id from emitted record shape (Relay: `edges[].node.id`); query-body parsing last resort.
- Do not raise the global BFS depth; keep GraphQL-specific handling in patternFinder.

**C8 · CF2 parity remainder — Ship D2 first; blocked on ~10 CF2 production samples**
- Pull ~10 CF2 exports joining configured traceKeyTemplate to one parsed record each; classify blanks into wrong-path vs eval-failure.
- Fix options after classification: trace-key-only `data` alias in main.js:201–228 (preferred) or nested-path guidance in UI. Never change baseHBDelegate globally — shared with webhook.successBody.
- Test matrix: `{{id}}`, `{{record.id}}`, `{{data.id}}`, `{{record.resource.id}}` against wrapped and unwrapped records, legacy and CF2.

**C9 · Reduce noisy URI segments — Ready (final word list reviewed in PR)**
- Lowercase each segment before the exclusion check (patternFinder.js:286–309).
- Starter exclusion set (conservative): api, rest, graphql, search, export, list, bulk, batch, data. Prefer deprioritizing ambiguous words over hard-excluding.
- Acceptance: 'API' and 'Search' no longer candidates; orders/customers unaffected.

**C10 · Explicit no-reliable-key state — Needs sign-off on reason-code enum (proposal below)**
- Add `getTraceKeyResult(record, template, traceKeyPattern, _flowId)` → `{ value, source: template|custom|map|dictionary|null, field, reason }`; getTraceKey delegates to it, string/null contract unchanged.
- Proposed enum: OK, TEMPLATE_EVAL_FAILED, TEMPLATE_EMPTY, NO_CANDIDATE_FIELD, CANDIDATES_NOT_UNIQUE, EMPTY_RECORD, TRUNCATED.
- Propagate: exportDataConverter page metadata → qgmw.js:186–190 / injectTraceKeys → preview `dataRecordTraceKeyDetails[]` → UI all-missing state.
- Reasons are metadata only — never stamp a sentinel value as the key.

---

## 5. Release Plan — Phases in Shipping Order

### Phase 1 · Correctness (go now · low risk)
1. D1 consumer + D2 + D5 — `tracekey-common/main.js` (publish first)
2. D1 producer + Change 3 + Change 9 — `tracekey/patternFinder.js` (after step 1)
3. D4 — `endpoint-service/responseTransformUtil.js` (after step 1 published)

### Phase 2 · Connector coverage (after Phase 1 · medium risk)
4. D3 — CF2 legacyId resolution
5. Change 1 + Change 2 — map expansion + known-connector detection (batched)

### Phase 3 · Template parity + GraphQL (needs Phase 1 prod data · medium risk)
6. Change 8 remainder — after D2 error telemetry proves cause
7. Change 7 — GraphQL resourcePath (parallel with step 6)

### Phase 4 · Batch framework (biggest build · high risk)
8. Change 5 + Change 6 — batch inference + uniqueness gate (one project)
9. Change 10 — reason metadata on 5/6 result API
10. Change 4 — widened fallbacks (only after step 8 gate exists)

**Dependency rules**
- Ship independently: D5, Change 9, Change 3, D2
- Ordered pairs: D1 consumer before D1 producer; D4 after D1 consumer
- Must batch: D3 + Change 1 + Change 2 | Change 5 + Change 6 | Change 4 + Change 6 | Change 10 follows 5/6
- Conditional: Change 7 fallback waits for Change 5; Change 8 waits for D2 telemetry
- Hard constraint (D6): every train publishes tracekey-common → tracekey → bumps identical pins across all consumers

| Phase | Risk | Defects | Changes | Goal |
|---|---|---|---|---|
| 1 · Correctness | Low | D1, D2, D4, D5 | 3, 9 | Fix bugs; improve URI inference |
| 2 · Connector coverage | Medium | D3 | 1, 2 | Map expansion; CF2 + known HTTP profiles |
| 3 · Parity + GraphQL | Medium | — | 7, 8 | Close CF2 template gap; GraphQL resourcePath |
| 4 · Batch framework | High | — | 5, 6, 10, then 4 | Inference, uniqueness gate, reason codes |

---

## 6. Expected Metric Impact by Phase

*Directional targets only — exact gains depend on connector traffic mix. Verify each phase: re-run workbook queries (no-key % and duplicate % per cohort) plus D2 template-failure metric, four weeks pre/post deploy.*

**After Phase 1**
- Coverage: small direct gain (single-digit points off 84.37% legacy no-template)
- Quality: fewer wrong keys from URI noise (Change 9)
- Measurement: D2 makes template-failure rate measurable; D4 makes preview match production

**After Phase 2**
- Coverage: biggest no-template lever — targeted connectors drop from 84.37% / 77.34% toward near-complete keys
- Duplicates: roughly flat — curated map fields, low collision risk

**After Phase 3**
- CF2 with-template no-key: 20.14% → low single digits (legacy-template 0.24% is the ceiling)
- GraphQL no-template: 7.42% → under ~3%
- CF2 template duplicates (9.50%): partial improvement via nested-path guidance

**After Phase 4**
- Coverage: batch inference catches long-tail APIs with non-standard ID fields
- Duplicates: uniqueness gate drives inferred-key duplicates (0.33%–10.79%) toward zero
- Explainability: Change 10 reason codes eliminate "unknown missing" bucket

---

## 7. Phase 1 Delivery — Three PRs, One Release Train

**PR 1** · `integrator-common-util` · `tracekey-common/lib/tracekey/main.js`  
D1 consumer · D2 · D5

**PR 2** · `integrator-common-util` · `tracekey/lib/tracekey/patternFinder.js`  
D1 producer · Change 3 · Change 9 *(Changes 3 and 9 are new feature code — not automatic from defect fixes)*

**PR 3** · `endpoint-service` · `responseTransformUtil.js`  
D4 — ships after PR 1 published

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

- **Jira:** IO-208154 (spike), IO-38405 (null over generated keys), IO-153375 (256-char cap, shipped)
- **Confluence:** 2290384905 (HTTP adaptor recommendations), 2167341069 (broader trace-key actions)
- **Evidence workbook:** Google Sheet "Summary w/ config from logs"
- **Main revisions:** integrator-common-util 078fe54 · integrator-adaptor 1a8be58 · http-adaptor c0318a7 · flow-execution 347705f · endpoint-service 00dc15a · em-util 1302bda · integrator-models 4743a6b · flow-management 670fb54 · integrator-workers c7839b9 · integrator-ui c287211
