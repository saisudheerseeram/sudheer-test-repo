# IO-69506 Spike: Custom XML Parse Strategy Does Not Evaluate Real XPath

Spike: [IO-69506](https://celigo.atlassian.net/browse/IO-69506)  
Story: [IO-37388](https://celigo.atlassian.net/browse/IO-37388)  
Epic: [IO-47804](https://celigo.atlassian.net/browse/IO-47804)

**Suggested implementation PR:** [celigo/http-adaptor#1975](https://github.com/celigo/http-adaptor/pull/1975)

---

## Summary

For HTTP exports and lookups with XML **Custom** parse strategy (`parserVersion` 1), real XPath on `resourcePath` / success path / paging paths returns **empty results with no error**.

The same expression works on **Automatic** (`parserVersion` 0) because Automatic uses a DOM + `xpath.select` engine.

Custom uses a streaming engine (`@celigo/parsers` XmlToJson) that only matches literal element paths such as `/data/item`. It does not evaluate `//`, `local-name()`, `text()`, predicates, wildcards, or attributes.

The two strategies share the same pipeline through `requestRobustly`. They diverge **inside** `XmlParser`, including on the success-path check that still runs *inside* `requestRobustly` after the HTTP response is back.

## Customer Impact

* Switching parse strategy from Automatic to Custom with a path like `//item` or `//*[local-name()='item']` makes preview and export pages show `0 Pages, 0 Records`.
* There is no `invalid_xpath` unless the expression is illegal for the DOM engine (for example `/data/item.text()` is JS-style, not XPath; the valid form is `/data/item/text()`).
* Success / fail path, resource path, and paging XPaths on Custom can all silently miss.
* Snowflake config population for this bug: **29** non-deleted XML HTTPExport docs with Custom + full XPath (21 export + 8 lookup).
* Live traffic is much smaller: **1 of 29** executed in the last 90 days. **10 of 29** are attached to a flow. **1 of those 10 flows is enabled.**

## Root Cause

The root cause is an engine split inside `http-adaptor/src/parsers/XmlParser.js`, decided by `commonUtil.getParserVersion(doc.parsers)`.

This was intentional for memory, not a UI bug. Custom was added so large vendor XML would not OOM the adaptor. Automatic kept full XPath 1.0.

### Automatic path

Automatic (`parserVersion` 0) has no `{ type: 'xml', version: 1 }` on `doc.parsers`.

It loads the full response body into a DOM (`xmldom-new`) and calls `xpath.select(path, doc)`.

That is real XPath 1.0, so `//item`, `*[local-name()='Worker']`, `text()`, `@attr`, and unions work.

JSON shape is verbose, for example `{ Name: [{ _: 'John' }] }`.

### Custom path

Custom (`parserVersion` 1) means `parsers` contains `{ type: 'xml', version: 1 }`.

Until the fallback runs, Custom never calls `xpath.select`. Flow:

* `formatSimplePath(path)` only prepends `/` if missing
* `XmlToJson.parseChunk(body)` compares the stream's current element path (for example `/data/item`) to `resourcePath`

That is **not** an XPath engine. It does string-prefix matching on element names.

| Syntax | Meaning | Streaming result |
|---|---|---|
| `//item` | descendant-or-self | no match |
| `//*[local-name()='item']` | any element named `item`, ignore namespace | unsupported |
| `/data/item/text()` | text node | unsupported |
| `/data/item[1]`, `@id`, `*`, `\|` | predicates, attributes, wildcards, unions | unsupported |

Simple `/a/b/c` happens to look like XPath and works. The UI labels the field "XPath" for both strategies, which is the mismatch.

Amazon SP (`simpleXpathsOnly`) always stays on streaming, even after this change.

### Why this is not a `requestRobustly` fork

Request build, signing, and the axios call are shared.

The first parser call can still happen **inside** `requestRobustly`:

* `isSuccessfulResponse`
* `containsPathAndValues`
* `parser.getNodes(successPath)`

If Custom + complex XPath is used as a success path, the empty result happens before `requestRobustly` callbacks.

Record extraction (`getResourcesFromResponse`), paging `getNodes`, and import `processSubmitResponse` use the same `XmlParser` methods **after** the call.

## Code Flow

### Common flow

1. Worker / agent sends `getNextPage` (export) or import / lookup submit.
2. `httpAdaptorCore.requestRobustly()` fills handlebars, signs, and makes the HTTP call.
3. After the body is back, `isSuccessfulResponse()` may call `parser.getNodes()` for success / fail path.
4. The adaptor then extracts records or paging tokens via `getResourcesFromResponse` / `getNodes`.

### Automatic flow

1. `getParserVersion` returns 0.
2. `getNodes` builds a DOM and runs `xpath.select`.
3. Nodes become verbose JSON.

### Custom flow (today, flag off)

1. `getParserVersion` returns 1.
2. `getNodes` / `getResourcesFromResponse` always take the streaming branch.
3. If the path is not a literal element path, the stream emits zero records.
4. Preview and the export page look empty. No exception.

## Additional Behavior Note: memory

Code in `getResourcesFromResponse` documents that ~25 MB of complex XML can expand to ~600 MB of DOM nodes in heap.

That is why Custom streaming exists. Supporting `//` on Custom means paying Automatic's heap cost **for those paths only**. Simple Custom paths should stay streaming.

## Permanent Fix Options

### Option 1: Hybrid DOM fallback for Custom + real XPath

Recommended. Implemented in [PR #1975](https://github.com/celigo/http-adaptor/pull/1975).

After classifying the path with `isSimplePath`:

* Custom + `/data/item` stays on streaming.
* Custom + `//item`, `local-name()`, `text()`, `@`, `*`, `[ ]` uses DOM `xpath.select`, then converts matched nodes with the **custom** parser config so JSON stays lean.
* Automatic stays DOM.
* Amazon SP `simpleXpathsOnly` never uses DOM.

The fallback is gated by `XML_CUSTOM_PARSER_XPATH_FALLBACK_ENABLED` (nconf, `env.yaml`, `devops_configured: true`). Default **off**.

Pros:

* Fixes the empty-results bug without putting a full XPath engine into the streamer.
* Simple Custom paths keep today's memory profile.
* Custom JSON stays lean. The 29 docs are not switched to Automatic's verbose shape.
* Flag-off merge is behavior-neutral for Custom XPath.

Cons:

* Custom + real XPath pays Automatic heap for that call. Large XML + `//` can OOM the same way Automatic already can.
* Needs staged enablement.

### Option 2: Teach streaming XmlToJson full XPath

Put descendant axes, predicates, and functions into `@celigo/parsers`.

Pros:

* Stays streaming for large files.

Cons:

* Large change in the wrong layer. Streaming XmlToJson is a path matcher, not an XPath engine.

### Option 3: Run all Custom through DOM

Pros:

* One engine. XPath always works.

Cons:

* Regresses memory for **231** simple-absolute Custom docs that are fine on streaming today.

## Production Impact of the 29 (as of 2026-08-27)

Source: Snowflake `SILVER.MONGODB.EXPORTS` (same Custom + full-XPath filter), `SILVER.MONGODB.JOBS` (last 90 days), `SILVER.MONGODB.FLOWS` (non-deleted).

Total XML HTTPExport docs: **19,442**. Custom + full XPath: **29**.

| Slice of the 29 | Count |
|---|---:|
| Config population | **29** (21 export + 8 lookup) |
| Attached to a non-deleted flow | **10** |
| Of those 10, flow is **enabled** | **1** |
| Of those 10, flow is **disabled** | **9** |
| Not on any live flow | **19** |
| Executed in the last **90 days** | **1** |

The 29 is a config count. Live fallback traffic today is **one enabled flow**.

**The one that ran**

* Export: Get employee terminations from Workday (`635849aa27114d57ab87d135`)
* Path: SOAP `/*[local-name()='Envelope']/.../'Worker'`
* Flow: Sync employee terminations from Workday to NetSuite (`635849aa27114d57ab87d13e`) — **enabled**, page generator
* 128 jobs in 90 days. Last job: 2026-08-26 20:06 UTC

The other 9 attached docs sit on disabled flows. Turning those flows back on would start using the fallback.

The two `//` exports (`//ReportInfo`, `//Read/Projecttask`) did **not** run in the last 90 days.

~14.6k simple-absolute Automatic exports are unaffected.

## Recommended Tests

Unit tests: `__tests__/unit/parsers/xmlParser.test.js` (IO-69506 block), including flag on/off.

QA preview (Custom, flag on):

1. `/data/item` — 2 lean records (streaming, unchanged)
2. `//item` — 2 lean records (was empty)
3. `//*[local-name()='item']` — same
4. `/data/item/text()` — valid XPath; **not** `/data/item.text()`
5. `count(//item)` — `[2]`
6. `/data/item.text()` — `invalid_xpath`

Flag off: `//item` empty again; `/data/item` still works.

Also re-run Automatic vs Custom on the same payload. Shape still differs: verbose vs lean.

## Backward Compatibility Considerations

The backward compatibility risk is real and should be treated carefully.

Users who are working correctly today are typically in one of these cases:

* They use Automatic, so XPath already works.
* They use Custom with a simple path `/a/b/c`, so streaming already matches.
* They use Amazon SP (`simpleXpathsOnly`), which stays on streaming.

The category this change is for:

* Custom parse strategy
* `resourcePath` / success path / paging path is real XPath (`//`, `local-name()`, etc.)
* They currently get empty results

That is the 29 docs. After the flag is on, those docs start returning records instead of empty. That is the intended fix, but it is still a behavior change.

### Risks if the fallback is enabled globally with no flag

Even though only Custom + complex paths change:

* A Custom + `//` export on a large XML body loads the full DOM (Automatic heap).
* A flow that was "working" with zero records (disabled, unused, or wrongly assumed empty) could start emitting records.
* Custom import `resourceIdPath` is still not evaluated per record. That is a pre-existing Custom contract and is out of scope.

### Examples

#### Example 1: Customer switched Automatic → Custom with `//item`

Today:

* Preview shows 0 records.
* They believe Custom is broken.

After flag on:

* Preview and export return lean records.
* This is the IO-69506 repro. Correct fix.

#### Example 2: Live Workday Custom SOAP export

Configuration:

* Custom parser
* `local-name()` Worker path
* Enabled flow: Sync employee terminations from Workday to NetSuite

Today:

* Streaming does not match the SOAP path, so pages can be empty.

After flag on:

* DOM evaluates the path. Records start flowing. Heap matches Automatic for that page (Workday Get_Workers pages in production are already in the ~8–10 MB Automatic range; this Custom flow's pages have been ~1 MB parsed JSON on similar OpenAir Custom traffic, but SOAP Worker XML can be larger).

#### Example 3: Simple Custom path `/AmazonEnvelope/.../Result`

Today and after flag on:

* Still streaming. No behavior change. This is the 231 simple-absolute Custom docs.

### Compatibility conclusion

A global Custom XPath behavior change without rollout control is not recommended.

The safest interpretation is:

* Automatic and Custom stay separate engines.
* Custom simple paths keep streaming by default.
* Custom real XPath should use DOM only when an explicit guard is on.
* Amazon SP stays streaming unconditionally.

### Recommended rollout approach

1. Keep Custom + simple path working with no change (no customer workaround required for `/a/b/c`).
2. Merge the hybrid fallback with the flag **off** (PR #1975).
3. Enable `XML_CUSTOM_PARSER_XPATH_FALLBACK_ENABLED` on QA. Confirm `//item` and `local-name()`.
4. Enable production. Watch heap on Custom + XPath flows. The only live production flow in the 29 today is Workday employee terminations.

Recommended default position:

* Do not change Custom XPath behavior by default.
* Use the flag for staged enablement.
* Do not force all Custom through DOM.

This preserves working Custom streaming configurations while allowing impacted XPath users to be unblocked when the flag is on.

## Current Workaround / Unblock

Until the flag is on:

* Switch the parse strategy back to **Automatic** if the path is real XPath. Automatic already evaluates `xpath.select`.
* Or change Custom `resourcePath` to a literal element path the streamer can match, for example `/data/item` instead of `//item`.
* Do not use JS-style `/data/item.text()`. Use `/data/item/text()`.

Workaround trade-off: Automatic returns verbose JSON and uses more heap. Custom simple paths stay lean and streaming.
