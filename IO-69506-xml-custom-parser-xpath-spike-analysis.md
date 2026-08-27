# IO-69506 · XML Custom Parser XPath Spike Analysis

Spike: [IO-69506](https://celigo.atlassian.net/browse/IO-69506)  
Story: [IO-37388](https://celigo.atlassian.net/browse/IO-37388)  
Epic: [IO-47804](https://celigo.atlassian.net/browse/IO-47804)

**Suggested implementation PR:** [celigo/http-adaptor#1975](https://github.com/celigo/http-adaptor/pull/1975) — hybrid DOM fallback for real XPath on custom XML parsers, gated by `XML_CUSTOM_PARSER_XPATH_FALLBACK_ENABLED` (default **off**).

---

## 1. Executive summary

Custom XML parse strategy (`parserVersion` 1) uses a **streaming** engine that only matches literal element paths (`/data/item`). Real XPath (`//item`, `//*[local-name()='item']`, `text()`, predicates) returns **empty results with no error**. Automatic strategy (`parserVersion` 0) uses a **DOM + xpath** engine and those expressions work.

That split was intentional (memory), not a UI bug. Supporting `//` on custom means paying Automatic's heap cost for those paths only.

Snowflake (`DATA_ROOM.MONGODB.EXPORTS` / `SILVER.MONGODB.EXPORTS`, non-deleted XML HTTPExport): **19,442** docs. The fallback would change behavior for **29 custom + full-XPath** docs (21 export + 8 lookup) **after the flag is on**. That 29 is a **config population**, not 29 live flows:

- **1 of 29** executed in the last 90 days (as of 2026-08-27).
- **10 of 29** are attached to a non-deleted flow.
- **1 of those 10 flows is enabled.** The other 9 attached flows are disabled. **19 of 29** are not on any live flow.

The ~14.6k simple-absolute Automatic exports are unaffected.

| Question | Answer |
|---|---|
| Why implemented this way? | Custom was built for large XML via streaming (`@celigo/parsers` XmlToJson). Streaming cannot evaluate real XPath; it does string-prefix matching on element paths. Automatic kept full XPath 1.0 via `xmldom-new` + `xpath`. |
| Why does `//` not work on Custom? | `//` is a descendant axis. The streaming engine treats the path as a literal `/`-separated name. `//item` does not match `/data/item` in the stream, so `getNodes` returns empty. |
| Impact if we support it? | Hybrid fallback: simple paths stay streaming; real XPath uses DOM. Memory for those paths matches Automatic (~25 MB XML → ~600 MB heap). Staged behind a flag. Config population at risk: **29** custom full-XPath docs. Live production traffic today: **1 enabled flow**. |

---

## 2. Why it was implemented that way

`XmlParser` has two engines. Dispatch is `commonUtil.getParserVersion(doc.parsers)`:

- **Automatic (`parserVersion` 0)** — no xml parser `version: 1` on the doc. Full document loaded into a DOM (`xmldom-new`). `xpath.select` evaluates XPath 1.0 (`//`, predicates, `local-name()`, `text()`, `@attr`, unions).
- **Custom (`parserVersion` 1)** — `parsers` contains `{ type: 'xml', version: 1 }`. Streaming `XmlToJson` from `@celigo/parsers`. Walks the document once, emits JSON when the current element path equals `resourcePath`. That is **not** an XPath engine.

Amazon SP (`simpleXpathsOnly`) always stays on streaming, even after this change.

**Why two engines existed**

1. **Memory.** Code in `getResourcesFromResponse` documents ~25 MB XML → ~600 MB of DOM nodes in heap. Streaming was added so large vendor XML (Amazon, Walmart, etc.) would not OOM the adaptor.
2. **Custom JSON shape.** Custom configs (`listNodes`, `includePaths`, `excludePaths`, lean JSON) live in the streaming parser. Automatic emits verbose `{ Name: [{ _: 'John' }] }`. Custom emits `{ Name: 'John' }`.
3. **UI mismatch.** The resource-path field is labeled "XPath" for both strategies. Automatic honors XPath. Custom only honors a subset that *looks* like XPath (`/a/b/c`). Users switching Automatic → Custom with `//item` see empty preview and empty export pages — the IO-69506 repro.

Related history: IO-30688, IO-29907, IO-28911, IO-45527, IO-45554, PRODUCT-1391.

---

## 3. Why `//` (and other real XPath) does not work on Custom

Until the fallback runs, Custom never calls `xpath.select`. Flow:

```
getNodes / getResourcesFromResponse
  if parserVersion === 1 || simpleXpathsOnly
    formatSimplePath(path)     // only prepends "/" if missing
    XmlToJson.parseChunk(body) // literal path match on currentPath
```

`XmlToJson` compares the stream's current element path (e.g. `/data/item`) to `resourcePath`. It does not implement:

| Syntax | Meaning | Streaming result |
|---|---|---|
| `//item` | descendant-or-self | no match — looks for a child named empty then `item` |
| `//*[local-name()='item']` | any element named `item`, ignore namespace | unsupported functions/predicates |
| `/data/item/text()` | text node | unsupported |
| `/data/item[1]`, `@id`, `*`, `\|` | predicates, attributes, wildcards, unions | unsupported |

**Silent empty, not an exception.** The stream finishes with zero records. Preview shows `0 Pages, 0 Records` without `invalid_xpath` unless the path is illegal for the *DOM* engine (e.g. `/data/item.text()` — JS-style, not XPath; use `/data/item/text()`).

Automatic works because `xpath.select('//item', doc)` is real XPath 1.0.

The two strategies share the same pipeline through `requestRobustly`. They diverge **inside** `XmlParser` on `parserVersion` / `simpleXpathsOnly`.

---

## 4. If we need to support it — what is the impact?

**Option chosen (PR #1975):** hybrid fallback, not "put XPath into the streaming engine" and not "run Custom entirely on DOM".

| Path | Engine after the change (flag **on**) |
|---|---|
| Custom + `/data/item` (simple) | Streaming — unchanged memory |
| Custom + `//item`, `local-name()`, `text()`, `@`, `*`, `[ ]` | DOM select, then convert nodes with the **custom** config so JSON stays lean |
| Automatic | DOM — unchanged |
| Amazon SP `simpleXpathsOnly` | Streaming — never DOM |

**Flag:** `XML_CUSTOM_PARSER_XPATH_FALLBACK_ENABLED` (nconf). Default **off** = merge is behavior-neutral. Enable per environment (QA first).

### Impact

| Area | Impact |
|---|---|
| Behavior (flag off) | None for Custom XPath (still empty). Scalar wrap (`count()` / `boolean()`) applies on DOM paths (Automatic); almost unused as `resourcePath`. `domRef` lazy-init crash fix is always on. |
| Behavior (flag on) | Config population of **29** custom full-XPath exports/lookups *would* start returning records instead of empty. Live traffic today is **1 enabled flow**. Simple Custom paths unchanged. |
| Memory / CPU | Only Custom + real XPath pays Automatic's DOM cost for that call. Simple Custom and Amazon SP do not. Large XML + `//` on Custom can spike heap the same way Automatic already can. |
| JSON shape | Custom stays lean; we do not switch those 29 to Automatic's verbose shape. |
| `resourceIdPath` on Custom import | Still not evaluated per record (pre-existing Custom contract). Out of scope. |
| Amazon SP | Exempt. |

### What we did **not** do

- Teach streaming XmlToJson full XPath (large, wrong layer).
- Force all Custom through DOM (would regress memory for the 231 simple-absolute Custom docs).

---

## 5. Production path inventory (Snowflake)

Source: `DATA_ROOM.MONGODB.EXPORTS`  
Filter: `deletedAt IS NULL`, `http.successMediaType = xml`, `adaptorType = HTTPExport`  
**Custom** = `parsers` contains `{ type: xml, version: 1 }`  
Path field: `http.response.resourcePath`  
Classification matches `isSimplePath` in `src/parsers/XmlParser.js`.

Total: **19,442** XML HTTPExport docs (exports + lookups).

### 5.1 Path types × strategy × does PR #1975 affect them?

| Path type | Strategy | Count | Example | Affected by PR? |
|---|---|---:|---|---|
| Simple absolute (`/a/b/c`) | automatic | **14,619** | `/AmazonEnvelope/Message/ProcessingReport/Result` | **No** — already DOM |
| Simple absolute | custom | **231** | `/AmazonEnvelope/Message/return_details` | **No** — still streaming |
| Simple relative (`a/b/c`) | automatic | **845** | `response/Read/Project` | **No** |
| Simple relative | custom | **4** | `s:Envelope` | **No** — still streaming (`:` is a simple segment) |
| Root (`/`) | automatic | **1,151** | `/` | **No** |
| Root | custom | **576** | `/` | **No** |
| Missing | automatic | **17** | *(empty)* | **No** |
| Missing | custom | **2** | *(empty)* | **No** |
| Other / invalid | automatic | **7** | `\`, spaces in path, `TBD - pending…` | **No** |
| Full XPath: `local-name()` | automatic | **1,493** | `/*[local-name()='Envelope']/…/'Worker'` | **No** — already works |
| Full XPath: `local-name()` | custom | **27** | same Envelope/Body/Worker path | **Yes if flag on** — empty today |
| Full XPath: `//` descendant | automatic | **407** | `//ReportInfo` | **No** |
| Full XPath: `//` descendant | custom | **2** | `//Read/Projecttask` | **Yes if flag on** |
| Full XPath: `text()` | automatic | **14** | `/RequestReportResponse/…/ReportRequestId/text()` | **No** |
| Full XPath: `text()` | custom | **0** | — | — |
| Full XPath: `*` wildcard | automatic | **42** | `/*` or `*` | **No** |
| Full XPath: `*` wildcard | custom | **0** | — | — |
| Full XPath: predicate `[1]` | automatic | **4** | `…/member[1]/value/i4` | **No** |
| Full XPath: `@attr` | automatic | **1** | `/SubmitOrder/Order[@Id]` | **No** |
| Full XPath: `@attr` | custom | **0** | — | — |

**Custom + full XPath = 29** (27 `local-name()` + 2 `//`). That is the population the flag is for.

Flag **off** (default): none of these rows change in production except the latent `domRef` crash fix and rare Automatic `count()`/`boolean()` success-path wrapping.

### 5.2 Custom + full XPath paths (the 29)

| Kind | Path | n |
|---|---|---:|
| export | `/*[local-name()='Envelope']/…/'Worker'` | 11 |
| lookup | same Worker path | 6 |
| export | `//*[local-name() = 'ShipmentData']/*[local-name() = 'member']` | 3 |
| export | `/*[local-name()='Envelope']/…/'getReportJobStatusResponse'` | 2 |
| lookup | same getReportJobStatusResponse | 1 |
| export | `/*[local-name()='response']/…/'shipment'` | 1 |
| export | `//Read/Projecttask` | 1 |
| export | `//ReportInfo` | 1 |
| lookup | pixiGetOrderHeader … `local-name()` chain | 1 |
| export | GetNewOrdersResponse … `WEB_ORDER` | 1 |
| export | `/*[local-name()='collection']/*[local-name()='SupplierConnectInvoice']` | 1 |

### 5.3 Are those 29 on a flow, and did they run? (as of 2026-08-27)

Sources: `SILVER.MONGODB.JOBS` (last 90 days, ~2026-05-29 → 2026-08-27) and `SILVER.MONGODB.FLOWS` (`deleted_at` is null). Flow attachment checked on `exportId`, `pageGenerators`, `pageProcessors`, `routers`, `childExports`, `exports`, and `runNextExportIds`. Lookups appear as a page-processor or router step with `_exportId`.

| Slice of the 29 | Count |
|---|---:|
| Config population (custom + full XPath) | **29** (21 export + 8 lookup) |
| Attached to a non-deleted flow | **10** |
| Of those 10, flow is **enabled** | **1** |
| Of those 10, flow is **disabled** | **9** |
| Not on any live flow | **19** |
| Executed in the last **90 days** | **1** |

The 29 is a config count. Live fallback traffic today is **one enabled flow**.

**The one that ran (and the only enabled flow)**

| | |
|---|---|
| Export | Get employee terminations from Workday (`635849aa27114d57ab87d135`) |
| Path | SOAP `/*[local-name()='Envelope']/.../'Worker'` |
| Flow | Sync employee terminations from Workday to NetSuite (`635849aa27114d57ab87d13e`) — **enabled**, page generator |
| Jobs in 90d | 128 |
| Last job | 2026-08-26 20:06 UTC |

**The other 9 attached docs sit on disabled flows** (turning the flow back on would start using the fallback):

| Doc | Kind | Flow | Slot |
|---|---|---|---|
| 103.1 Shipments | export | 103.0 Dotcom to NetSuite - Shipments | page generator |
| GET Employees from WORKDAY | export | WORKDAY (SOAP): Employees | page generator |
| Get Amazon delivered shipments | export | Amazon Shipments to SAP S/4 HANA Fulfillments | page generator |
| Get OpenAir project tasks (`//Read/Projecttask`) | export | OpenAir project tasks to Jira Cloud platform issues | page generator |
| Get employees from Workday | export | Sync Employee Updates from Workday to NetSuite | page generator |
| RobertTest | export | New flow | page generator |
| Workday Ed SOAP | export | PoC Workday Ed SOAP | page generator |
| Fetching the employee details | lookup | OLD FLOW - WorkDay workers to Litmos User | router |
| Marian - Pixi SOAP Order Header | lookup | Celigo PoC Pixi -> Emarsys Emails | page processor |

The two `//` exports (`//ReportInfo`, `//Read/Projecttask`) did **not** run in the last 90 days. `//Read/Projecttask` is on a disabled flow; `//ReportInfo` is not on a live flow.

Lookups often do not get their own `JOBS` row (they run nested in an import). Splunk NA last 30 days also had 0 hits on the idle lookup IDs.

### 5.4 Full XPath flavor mix (all strategies, 1,990 docs)

Mostly SOAP `local-name()` and `//` descendants. Almost no `text()` / `@attr` on resource path.

| Flavor | Approx. count |
|---|---:|
| `local-name()` + `*` (no `//`) | ~1,108 |
| `//` + `local-name()` | ~410 |
| `//` only (e.g. `//ReportInfo`) | ~407 |
| `*` only | ~42 |
| `text()` | ~16 |
| `@attr` | ~2 |

This inventory is **resourcePath only**. Success/fail paths are a separate field and were not counted here.

---

## 6. How to test

Unit tests: `__tests__/unit/parsers/xmlParser.test.js` (IO-69506 block), including flag on/off.

QA preview (Custom, flag on):

| Resource path | Expected |
|---|---|
| `/data/item` | 2 lean records (streaming) |
| `//item` | 2 lean records (was empty) |
| `//*[local-name()='item']` | same |
| `/data/item/text()` | valid XPath; **not** `/data/item.text()` |
| `count(//item)` | `[2]` |
| `/data/item.text()` | `invalid_xpath` (illegal syntax) |

Flag off: `//item` empty again; `/data/item` still works.

Also re-run Automatic vs Custom on the same payload (shape still differs: verbose vs lean).

---

## 7. Suggested PR

**https://github.com/celigo/http-adaptor/pull/1975**

- Branch: `IO-69506-xml-custom-xpath-fallback` from `main`
- Flag: `XML_CUSTOM_PARSER_XPATH_FALLBACK_ENABLED` in `env.yaml` (`devops_configured: true`)
- Decision record in http-adaptor: `docs/decisions/2026-08-26-xml-custom-parser-xpath-hybrid-fallback.md`

Rollout: merge with flag unset → enable on QA → confirm `//item` / `local-name()` → enable production. Watch heap on XML-heavy Custom + XPath flows after enable. The only live production flow in the 29 today is Workday employee terminations (SOAP `local-name()` Worker path).
