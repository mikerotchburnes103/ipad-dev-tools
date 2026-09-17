# Tampermonkey Compatibility Evaluation

**Target evaluated:** `/home/user/ipad-devtools-source.js`, the generated minified artifact, and the packed bookmarklet.

**Evaluation basis:** static review against the attached `tampermonkey-compat-test 2.md` plan, plus focused synthetic checks for userscript execution, persistent request rules, trusted updates, rollback history, and packed-bookmarklet behavior.

## Executive assessment

The bookmarklet is a capable **page-context userscript runner**, but it is not yet a drop-in Tampermonkey replacement.

It is strongest for:

- `@grant none` scripts that run after the page has loaded.
- Metadata parsing and URL matching.
- Same-origin persistent script values.
- `@require` and `@resource` loading when browser CORS allows the fetch.
- Menu-command presentation inside the Userscripts panel.
- Page-side fetch/XHR observation and rewriting.
- CSP fallback when the page allows an inline script nonce or inline script execution.

It is weakest for features that require extension privileges or pre-page timing:

- True `document-start` execution.
- Cross-origin `GM_xmlhttpRequest` without CORS.
- Popup-resistant `GM_openInTab`.
- Full Tampermonkey sandbox isolation.
- Strict per-`@grant` permission enforcement.
- Automatic execution inside iframes and SPA navigations.

## Result legend

- **Pass** — implemented behavior meets the test-plan expectation, subject to normal browser permissions.
- **Partial** — implemented with a documented bookmarklet/browser limitation.
- **Fail** — the requested compatibility behavior is absent or materially different.
- **Not tested** — requires a real browser or regression corpus and was not established by the current checks.

## Section results

| Section | Result | Assessment |
|---|---|---|
| 1. Metadata parsing | **Partial** | Core directives work and malformed local metadata is rejected, but some `@include` semantics differ. |
| 2. GM_* API surface | **Partial** | Many APIs exist and are now filtered by declared grants; privileged cross-origin behavior and browser-controlled popup/clipboard behavior remain limited. |
| 3. Sandbox and scope | **Partial** | Privileged scripts use wrappers and page-context scripts are wrapped in an IIFE, but a bookmarklet cannot provide a separate browser isolated world. |
| 4. Timing and injection | **Partial** | Document-end, document-idle-after-load, dependency ordering, and navigation re-matching are improved; true document-start execution remains impossible from a bookmarklet. |
| 5. Bookmarklet constraints | **Partial** | Relaunch protection, SPA re-matching, and CSP diagnostics exist; true pre-page execution and cross-origin extension storage do not. |
| 6. Real userscript regression | **Not tested** | A curated public-script corpus still needs to be run in Safari and at least one Chromium browser. |
| 7. GM.* Promise API | **Partial** | Promise APIs are present and use real Promises, but extension-level privilege and exact response behavior still need browser tests. |
| 8. Network/resource edge cases | **Partial / Fail** | Resource loading and binary response paths exist; privileged CORS and some policy behavior cannot be supplied by a bookmarklet. |
| 9. DOM/rendering interaction | **Partial** | Page DOM access works, but iframe injection, shadow-root styling, and mutation timing are not equivalent to Tampermonkey. |
| 10. Multi-script isolation | **Partial** | One failing script should not stop other scheduled scripts, and value storage is script-keyed; page-context monkey patches can still cross scripts. |
| 11. Security boundary | **Partial / Fail** | Local storage is origin-scoped and GM APIs are not placed on `window`, but `unsafeWindow` and other APIs are not filtered strictly by declared grant. |
| 12. Performance/resource cost | **Not tested** | No repeat-load memory or timing benchmark has been run. |
| 13. Cross-browser matrix | **Not tested** | The implementation targets iPad/Safari, but the attached matrix has not been populated. |
| 14. Reporting | **Partial** | The compatibility report documents several limitations, but the full case-by-case matrix remains outstanding. |

## Detailed findings

### 1. Metadata block parsing

**Passes:**

- Unicode metadata values are preserved as JavaScript strings.
- Multiple `@match`, `@include`, `@exclude`, `@exclude-match`, `@grant`, `@require`, and `@resource` entries are retained.
- Exclusion rules are checked before inclusion rules.
- `@version` is stored and used for update comparison, not execution.
- `@noframes` is recognized and prevents execution when the runner is already inside an iframe.

**Partial:**

- Matching supports wildcard patterns and slash-delimited regular expressions, but does not claim full Tampermonkey `@include` compatibility.
- A malformed local file is rejected and logged, but a raw URL without metadata is intentionally imported with generated current-site metadata. That is useful behavior, but it is not the strict “skip every missing metadata block” policy in the test plan.
- `@run-at document-start` is accepted but explicitly degraded because a bookmarklet is launched after page parsing has already begun or completed.

### 2. GM_* API surface

Implemented API groups include:

- Value storage and change listeners.
- Styles and element creation/removal.
- Menu commands.
- Clipboard, logging, notifications, downloads, tabs, resources, cookies, audio, and `GM_info`.
- Callback and Promise-style XHR wrappers.
- The newer `GM.*` value, resource, notification, download, tab, cookie, audio, and menu-command forms.

Important limitations:

- `GM_xmlhttpRequest` uses browser `fetch` with CORS and `@connect` checks. It cannot perform Tampermonkey’s privileged cross-origin requests when the remote server does not permit CORS.
- `GM_openInTab` uses `window.open` first and now provides a user-tappable retry notice when Safari blocks the popup; it still cannot guarantee an extension-style tab.
- `GM_notification` uses a real notification when permission is granted and now falls back to a dismissible in-page notification when it is not.
- `GM_setClipboard` depends on Safari clipboard permission or the fallback copy mechanism.
- The API object is filtered by exact legacy or modern grant operations. A script granted only `GM_setValue` receives that operation but not `GM_getValue` or `unsafeWindow`.
- `@grant none` does not receive the runner’s GM API bundle, which is correct for the no-grant path.

### 3. Sandbox and scope

The runner uses two execution paths:

- `@grant none` or `@unwrap`: indirect page evaluation.
- Privileged scripts: a wrapper function receiving the API values as parameters.

The wrapper prevents ordinary local variables in privileged scripts from becoming parameters or properties of the wrapper itself. Page-context and `@grant none` scripts are now wrapped in an IIFE while still executing against the real page global, so ordinary local declarations do not leak accidentally; explicit `globalThis`/`window` writes still affect the page by design.

`unsafeWindow` is now included only when the script declares the matching grant.

### 4. Timing and injection order

**Working:**

- `@require` files are fetched and resolved before the userscript body executes.
- A failed dependency rejects the execution path and does not silently run the body.
- `document-end` uses DOM readiness/`DOMContentLoaded` scheduling.
- Multiple matching scripts are scheduled in stored order.
- `document-body` has a body-availability observer.

**Bookmarklet limitations:**

- A bookmarklet cannot run before page scripts that already executed. `document-start` is therefore necessarily late, although it now runs immediately when the bookmarklet is invoked rather than adding another timer delay.
- `document-idle` now waits for the page `load` event when necessary and then uses `requestIdleCallback` or a timeout. It remains best-effort rather than exact extension timing.

### 5. Bookmarklet-specific constraints

**Implemented:**

- Re-injecting the bookmarklet is guarded by the `__ipad_devtools__` marker.
- The UI displays compatibility notes for late `@run-at`, unsupported grants, `@require`, resources, and page-context injection.
- The runner has a CSP fallback that tries a nonce-bearing inline script element and reports a clear error when the page blocks it.
- Values persist in local storage for the current origin.

**Missing or partial:**

- The runner now observes SPA `pushState`, `replaceState`, `popstate`, and `hashchange` changes and schedules enabled matching userscripts for the new URL. It still cannot survive a full navigation without being invoked again.
- A bookmarklet cannot persist across a full navigation without being invoked again.
- Storage is origin-scoped. A script’s values are not automatically shared across different subdomains/origins in the way a browser extension’s script storage can be.

### 6. Real userscript regression set

No public-script regression set has been run for this evaluation. This remains the most important practical validation step after synthetic API tests.

Recommended first corpus:

1. One simple `@grant none` DOM enhancer.
2. One script using `GM_setValue` and `GM_getValue`.
3. One script using `GM_addStyle`.
4. One script registering a menu command.
5. One script using `@require`.
6. One script using `GM_xmlhttpRequest` with `@connect`.
7. One script using `GM.*` Promises.
8. Auto-Duolingo, with its CSP and dynamic-evaluation limitations recorded separately.

### 7. GM.* Promise API

The Promise-based methods are implemented with actual `Promise.resolve(...)` or Promise wrappers, so `await` and `.then()` should work for the covered API groups.

The remaining compatibility concerns are:

- Exact Tampermonkey response shape for every `GM.xmlhttpRequest` response mode.
- Whether browser CORS allows the request.
- How scripts branch on `GM_info.scriptHandler`. The runner correctly reports `iPad DevTools`, not `Tampermonkey`; this avoids falsely claiming extension privileges but may expose different script branches.

### 8. Network and resource loading

**Working or partially working:**

- `@connect` is consulted for cross-origin `GM_xmlhttpRequest`.
- Blob and ArrayBuffer response paths are present.
- AbortController-backed timeouts are present.
- Fetch redirects are followed and `response.url` is returned as `finalUrl`.
- `@require` and `@resource` files are fetched before execution rather than being loaded as uncontrolled external script tags.
- The network monitor now preserves `Request` objects, including method, body, and headers. This is especially relevant to Auto-Duolingo.

**Not equivalent to Tampermonkey:**

- The bookmarklet cannot bypass CORS for `GM_xmlhttpRequest`.
- Timeout and abort callbacks need browser-level tests for exact event ordering.
- The page’s CSP can still block execution of fetched code after it has been retrieved.

### 9. DOM and rendering interaction

- Userscripts execute against the real page DOM and can query open shadow roots if their own code does so.
- Closed shadow roots remain inaccessible.
- `GM_addStyle` appends a document-level style element; it does not automatically style content inside a component’s shadow root.
- `@noframes` is honored when the runner is active inside an iframe, but the bookmarklet does not automatically inject itself into every same-origin iframe.
- Client-side rendering and node-removal races remain the userscript author’s responsibility. The runner does not add a universal mutation retry layer.

### 10. Multi-script isolation stress

The scheduler runs each script independently, so a synchronous exception in one script should be logged without preventing later scripts from being scheduled.

Value storage is keyed by the imported script ID, so two scripts do not accidentally share the same value object unless they share an ID through an explicit import/backup scenario.

The main limitation is page context: `@grant none` scripts and page-injected fallback scripts can intentionally monkey-patch shared globals such as `fetch`, prototypes, and DOM APIs. The runner now keeps ordinary top-level declarations inside an execution wrapper, but it does not provide extension-style isolated worlds.

Menu commands are stored per script and exposed when the corresponding script is selected in the Userscripts panel. They are not currently presented as one global browser context-menu list.

### 11. Security boundary

**Positive:**

- Userscript values use origin-scoped local storage rather than a cross-origin remote store.
- GM APIs are passed into privileged wrappers rather than being intentionally placed on the page’s global object.
- Remote resource text is not automatically evaluated as JavaScript.
- The runner has explicit confirmation for persistent network block/redirect rules and trusted update behavior.

**Current status:**

The runner filters the API bundle against the script’s declared legacy and modern grants. `unsafeWindow`, `GM_info`, legacy `GM_*` methods, and modern `GM.*` operations are passed when the matching operation is declared. For compatibility with existing public scripts such as DuoHacker that use `unsafeWindow` without declaring it, a narrowly detected implicit-usage exception is enabled and logged as a warning. This improves compatibility while making the deviation visible; it cannot stop a page-context script from explicitly modifying the page through `window` or `globalThis`.

### 12. Performance and resource cost

No benchmark has been run for:

- Cold-start to first userscript line.
- 50+ DOM nodes or large pages.
- Ten repeated userscript runs across page loads.
- Detached listeners or DOM retention.
- Large or binary `GM_xmlhttpRequest` responses.

The implementation does clean up runner-managed commands, listeners, resources, web-request rules, and automatic-run keys when scripts are disabled or replaced. That is positive, but it does not prove that arbitrary page-context timers and listeners are reclaimed.

### 13. Cross-browser matrix

The matrix is currently unpopulated. The implementation is aimed at iPad/Safari, but the following need real-browser confirmation:

- Safari `javascript:` URL restrictions.
- Clipboard permission behavior.
- Popup blocking for `GM_openInTab`.
- Strict CSP and nonce behavior.
- CORS behavior for `GM_xmlhttpRequest`.
- `requestIdleCallback` availability and timing.
- Local-storage behavior in private browsing and embedded contexts.

## Recommended next fixes

### Remaining priority 1 — browser privilege limits

These cannot be fully implemented by an ordinary bookmarklet without an extension or a trusted external service:

- Privileged cross-origin reads for `GM_xmlhttpRequest` when the server does not allow CORS.
- Extension-style tab creation from `GM_openInTab` (the bookmarklet intentionally copies the URL instead).
- Execution before page scripts for `document-start`.
- Extension-wide storage shared across unrelated origins.

The implementation now reports or offers a user-mediated fallback for these cases rather than silently pretending they are equivalent.

### Remaining priority 2 — real-browser validation

- Run the regression corpus in Safari/iPadOS.
- Populate the browser matrix.
- Benchmark cold-start, repeated injection, and long-running userscript memory behavior.
- Test exact `GM_xmlhttpRequest` callback ordering and popup/clipboard permissions.

### Priority 3 — browser test corpus

Run the attached plan in Safari on iPad, then repeat a smaller smoke set in Chrome/Firefox/Edge. Record results as Pass, Partial, or Fail instead of inferring extension behavior from API presence.

## Current conclusion

The project is ready to describe itself as:

> An iPad/Safari bookmarklet with a practical Tampermonkey-compatible userscript runner for page-context scripts, selected GM APIs, persistent values, resources, runtime controls, and CSP-aware fallbacks.

It should not yet be described as:

> A full Tampermonkey replacement.

The highest-value remaining work is a real Safari/iPadOS regression run using the attached test plan, followed by filling the cross-browser and performance sections with measured results.
