# iPad DevTools Bookmarklet

A self-contained, iPad/Safari-friendly developer-tools panel that runs as a bookmarklet on the current web page.

It provides a practical subset of browser DevTools without requiring an App Store extension, Safari Web Extension, server, CDN, or build step at runtime.

> This is a bookmarklet, not a full browser extension. Safari page, popup, Content Security Policy, CORS, storage, and device-management restrictions still apply.

## Features

### Console

- Run JavaScript against the current page.
- Plain Enter-to-run remains the intentional console shortcut.
- Syntax highlighting, autocomplete, command history, previous-state restoration, and a compact rerun control.
- Save, load, rename, duplicate, import, export, and delete reusable snippets.
- Copy individual log entries or the complete console log.

### Elements and styles

- Inspect the page DOM in a compact tree.
- Search through page elements and text.
- Pick an element directly from the page.
- Edit an element as HTML, delete it, and undo or redo changes.
- Replace text throughout the page and optionally watch for newly added matching content.
- Inspect and edit inline styles.
- Highlight the selected element while scrolling.
- Run a basic accessibility report.

### Network

- Capture page-side `fetch` and `XMLHttpRequest` activity.
- Filter requests by status and search by URL or request data.
- Inspect request details.
- Copy requests as Fetch code or cURL commands.
- Replay requests where the page permits it.
- Add persistent per-site URL rules to block or redirect future page-side requests.

Network rules are not Safari Content Blocker rules. They do not provide browser-level interception for every request, navigation, media load, service worker, or extension-managed network operation.

### Storage and page state

- Inspect and edit `localStorage` and `sessionStorage`.
- Inspect readable cookies.
- Add, edit, delete, or clear storage values.
- Copy the current page URL.
- Copy a page-state snapshot or export logs.
- Capture a page screenshot where the browser permits canvas-based capture.

DevTools settings, saved snippets, userscripts, update history, network rules, and previous panel state are stored in the current site's `localStorage`.

### Sources and performance

- Inspect available page sources and refresh the source list.
- View basic page performance metrics.

### Tampermonkey-compatible userscripts

The **User scripts** tab supports a bookmarklet-compatible userscript workflow:

- Import a `.user.js` file.
- Import a metadata or raw JavaScript URL through the visible URL form.
- Create and edit a new userscript.
- Enable, disable, delete, export, save, and run scripts.
- Run enabled scripts that match the current page.
- Run a selected script manually when necessary.
- Load `@require` and `@resource` files through browser-permitted fetches.
- Register userscript menu commands, value storage, styles, elements, notifications, clipboard actions, and supported GM APIs.
- Check updates from declared `@updateURL` or `@downloadURL` values.
- Automatically install trusted declared-source updates when automatic updates are enabled.
- Ask for confirmation before installing other update sources.
- Keep up to five rollback copies when an update replaces an active script.
- Copy the generated URL for `GM_openInTab` instead of attempting a popup that Safari may block.

The default update mode is power-user oriented: enabled userscripts with declared update URLs are checked when the bookmarklet starts, and trusted declared-source updates can be installed automatically. This can be changed in Settings.

The runner applies declared-grant filtering. It also provides a narrow compatibility exception for scripts that actually reference `unsafeWindow` without declaring `@grant unsafeWindow`; this exception is logged visibly.

## Installation on iPad Safari

1. Download or open [`ipad-devtools-bookmarklet.txt`](ipad-devtools-bookmarklet.txt).
2. Copy the complete single-line value, including the `javascript:` prefix.
3. In Safari on iPad, create any temporary bookmark.
4. Open Safari's bookmarks, edit that bookmark, and replace its URL with the copied bookmarklet text.
5. Save the bookmark.
6. Open the page you want to inspect and tap the bookmarklet bookmark.

The panel appears at the bottom of the page. Use the drag handle to resize it. The **Collapse** button leaves a small bottom bar, and **Close** removes the panel for the current page.

If Safari strips the `javascript:` prefix while editing the bookmark, restore the prefix before saving. The bookmarklet must remain one line.

## Repository files

| File | Purpose |
| --- | --- |
| `ipad-devtools-source.js` | Readable development source. |
| `ipad-devtools-min.js` | Minified JavaScript artifact. |
| `ipad-devtools-bookmarklet.txt` | Packed, minified `javascript:` bookmarklet for Safari. |
| `tampermonkey-compatibility-evaluation.md` | Detailed compatibility evaluation and known limitations. |
| `userscript-compatibility-report.md` | Userscript API and regression notes. |

For normal iPad use, install the `.txt` bookmarklet artifact. The source file is intended for review and development.

## Development

The implementation is intentionally self-contained. The source uses vanilla JavaScript, DOM APIs, Shadow DOM, browser storage, and page-side browser APIs. There is no runtime dependency on a framework or external stylesheet.

When changing the source:

1. Edit `ipad-devtools-source.js`.
2. Run a JavaScript syntax check.
3. Rebuild `ipad-devtools-min.js`.
4. Rebuild `ipad-devtools-bookmarklet.txt` from the minified artifact.
5. Validate both the readable source and packed bookmarklet in a browser or DOM test harness.
6. Commit all three artifacts together so the hosted bookmarklet does not lag behind the source.

A bookmarklet cannot reproduce extension-only capabilities. Test changes on the target iPad/Safari version, especially for:

- Clipboard permission and user-gesture requirements.
- Popup and download policies.
- Strict page CSP and nonce handling.
- Cross-origin fetch and `@connect` behavior.
- Private browsing and storage limits.
- Embedded frames and MDM-managed Safari restrictions.
- Pages that replace their DOM or use service workers.

## Compatibility boundaries

This project deliberately does not claim full Tampermonkey or Safari Web Extension equivalence.

- `@run-at document-start` cannot run before a bookmarklet is tapped and injected.
- Background execution and extension-wide script storage are unavailable.
- `GM_xmlhttpRequest` is implemented with page-permitted requests, so CORS, `@connect`, credentials, and server policy apply.
- `GM_webRequest` affects future page-side requests observed by the compatibility layer; it is not browser-level request interception.
- `GM_cookie` can only access cookies exposed through `document.cookie`; HttpOnly cookies are not available.
- Isolated worlds from `@inject-into` and `@sandbox` cannot be created by a normal bookmarklet.
- Download, notification, clipboard, and navigation behavior remains subject to Safari permissions.
- The bookmarklet does not modify or bypass third-party authentication, API-key, or account systems used by page applications such as Auto-Duolingo.

## Privacy and security

The bookmarklet executes with access to the current page. Only install userscripts and run code that you trust. A userscript can modify the page and may be able to read data exposed to page JavaScript.

The tool does not require a remote backend. It stores its own settings and saved data in the current site's browser storage. Network capture records requests observed by the page-side hooks; use the Storage and Settings controls to clear saved data when required.

## License

Add the project's chosen license before publishing the repository. Until a license is added, GitHub users should treat the source as available for viewing but not automatically licensed for redistribution or modification.
