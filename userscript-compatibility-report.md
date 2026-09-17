# iPad DevTools userscript compatibility report

**Date:** 2026-09-15

## Result

The previous 62/62 API-surface result was produced before the new strict per-`@grant` filtering and is not being carried forward as a current full-suite claim. The current implementation has passed focused source-and-packed smoke tests for:

- Strict grant filtering.
- SPA URL-change re-matching.
- Trusted update installation and rollback history.
- Persistent request blocking.
- Packed bookmarklet startup.
- Request-body/method preservation.

The complete 62-check browser-style harness should be rerun after the permission changes before assigning a new API denominator.

The focused tests ran against both the readable source and the current packed bookmarklet.

## Passed areas

- `@require` execution and ordering, including separate dependency/body scopes
- `@resource` text and URL access
- Initial userscript execution is not blocked by slow or failing background update checks
- `@updateURL` metadata-only responses are followed by the declared full `@downloadURL` before installation
- Duolingo DuoHacker (`@grant GM_xmlhttpRequest`, `@grant GM_addStyle`) reaches completed userscript execution in both source and packed smoke tests; its undeclared `unsafeWindow` usage is reported and supported for compatibility
- `GM_getValue`, `GM_setValue`, `GM_deleteValue`, `GM_listValues`
- Bulk value APIs: `GM_getValues`, `GM_setValues`, `GM_deleteValues`
- Value-change listeners
- `GM_addStyle`, `GM_addElement`, `GM_removeElement`
- `GM_openInTab`
- `GM_setClipboard`
- `GM_notification`
- `GM_log`
- Menu commands: register and unregister
- `GM_download`
- `GM_xmlhttpRequest`
- `GM_webRequest`
- Tab APIs: get, save, and list
- Browser information
- Cookie list, set, and delete
- Audio mute/state APIs
- `GM_info`
- `unsafeWindow`
- Promise namespace `GM.*`
- `window.close`, `window.focus`, and `window.onurlchange`
- `@match` page execution
- `@unwrap` page-context execution with ordered `@require` code (separate end-to-end check)
- CSP-safe script-element fallback when unsafe-eval is blocked, including a granted userscript (separate end-to-end check)
- No-eval packed bootstrap with page nonce support (separate end-to-end check)
- Best-effort `@run-at` scheduling for document-start, document-body, document-end, and document-idle (separate implementation check)
- Currently-running userscript panel with per-script and disable-all controls (separate end-to-end check)
- Launch-time checks for declared userscript update URLs with automatic trusted-update installation and rollback history (separate end-to-end check)
- Site-persistent manual network block and redirect rules with explicit confirmation (separate end-to-end check)
- Visible URL import form with raw-text import (separate end-to-end check)
- Transient URL fetch retry and clear HTML/CORS error handling (implementation check)
- Experimental user-script key helper detection and Shift keydown/keyup dispatch (separate end-to-end check)
- Compact movable key dock rendering (separate end-to-end check)
- Right-click, Alt, Control, letter, and number key detection/dispatch (separate end-to-end check)

## Important interpretation

Any future numeric score should be treated as a runtime API pass ratio, not proof that a bookmarklet has the same privileged capabilities as Tampermonkey, Violentmonkey, Greasemonkey, or a Safari extension. The key-helper check and the new navigation/update/network tests are additional end-to-end checks and should be reported separately from the legacy API denominator.

Known intentional limitations remain:

- Bookmarklet execution occurs after the page has loaded; `@run-at document-start` cannot be reproduced.
- `@inject-into` and `@sandbox` do not create an extension-style isolated world.
- `GM_xmlhttpRequest` is implemented through page-permitted `fetch`, so CORS, `@connect`, cookies, and server policy still apply.
- `GM_webRequest` affects future page-side fetch/XHR calls only; it is not browser-level request interception.
- `GM_cookie` can access only non-HttpOnly `document.cookie` values.
- `GM_openInTab` now copies the generated URL instead of opening a popup, avoiding Safari popup blocking; clipboard permission still applies. `GM_download` and notifications remain subject to Safari download and permission rules.
- Background execution, privileged extension tabs, and administrator-controlled Safari restrictions cannot be emulated.

The focused jsdom harness emitted no functional failures. Real Safari/iPadOS checks remain necessary for popup, clipboard, CORS, CSP, iframe, and timing behavior.
