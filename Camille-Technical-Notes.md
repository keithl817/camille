<img src="https://assets.cdn.filesafe.space/e3LOYAHOBn0PFjgBHsPS/media/69fbcdf4533d641a893d0934.webp" alt="Camille — Visual Composition for the AI-Built Web" width="600">

# Camille v1.2 — Technical Notes

*Visual Composition for the AI-Built Web*

A handover doc for future maintenance and extension. The intended reader is "future Claude or another developer adding a feature without re-deriving every design decision."

> Live at **[camille.sypmedia.com](https://camille.sypmedia.com)** · Copyright © 2026 SYP Media LLC. All rights reserved.

## Project layout

Single file: `index.html`. Approximately 1900 lines.

Self-contained — no build step, no `node_modules`, no separate CSS or JS files. Structure:

```
<!DOCTYPE html>
<html>
<head>
  <title>...</title>
  <style>...</style>          (about 400 lines: app chrome, toolbar, panel, image controls)
  <link rel="stylesheet"...>  CodeMirror CSS via cdnjs
  <link rel="stylesheet"...>  Dracula theme
  <script src="..."></script> CodeMirror core
  <script src="..."></script> xml, javascript, css, htmlmixed modes
  <script src="..."></script> closetag, matchbrackets, search, dialog addons
</head>
<body class="view-page">
  <header>...</header>
  <div id="toolbar">...</div>
  <main>
    <iframe id="editor-frame"></iframe>
    <textarea id="source-view"></textarea>  (replaced by CodeMirror at init)
    <div id="welcome">...</div>
    <aside id="props-panel">...</aside>
  </main>
  <div id="image-toolbar">...</div>          (floating, position: fixed)
  <div id="paste-modal">...</div>            (modal overlay)
  <div id="toast"></div>
  <script>(function() { ... })();</script>   (about 1000 lines: all logic)
</body>
</html>
```

Everything ships from one file. No mounting, no router, no framework.

## High-level architecture

Three editing surfaces, switched via a body class:

1. **Iframe with `designMode = 'on'`** — the visual editor. Each load of a document calls `frame.srcdoc = wrappedHtml` which triggers a clean re-load and `frame.onload` fires our handlers.
2. **CodeMirror instance** — the source editor. Initialized once via `CodeMirror.fromTextArea(textareaEl, opts)`. Visible via CSS based on the body class.
3. **Element Styles aside** — a 300px right sidebar inspector. Independent of view mode, toggleable.

Plus a floating image toolbar (`position: fixed`) that appears when an image is clicked.

## State

```js
let state = {
  filename: 'untitled.html',
  isFragment: false,
  view: 'page',            // 'page' | 'source' | 'split'
  loaded: false,
  fileHandle: null         // FileSystemFileHandle if Save-As'd
};
```

Other state vars (still inside the IIFE):

| Variable | Purpose |
|---|---|
| `widePreview: bool` | Affects `wrapFragment` only |
| `wrapEnabled: bool` | CodeMirror `lineWrapping` |
| `selectedEl: Element` | Element under the Styles panel pointer |
| `selectedImg: HTMLImageElement` | Currently selected image (for image toolbar) |
| `lastIframeRange: Range` | Text selection cached for selection-aware color application |
| `sourcePositions: Array<{tag, line}>` | Source-line mapping for click-to-locate |
| `cm: CodeMirror.Editor` | Source editor instance (may be null if CDN failed) |
| `panelOpen: bool` | Whether the Styles panel is visible |
| `activeLineHandle / activeGutterLine` | Current click-to-locate highlight |

## Initialization order (matters)

The IIFE runs top-to-bottom. The order of declarations matters because `let` is in TDZ before its line. Current order:

1. DOM element refs (`frame`, `sourceView`, etc.)
2. State object
3. `wrapEnabled` and `widePreview` (read from localStorage)
4. CodeMirror init via `fromTextArea`
5. `getSourceText` / `setSourceText` helpers
6. Wrap toggle wireup
7. Wide toggle wireup
8. Toast helper
9. `isFullPage` / `wrapFragment` (use `widePreview`)
10. `loadHTML` (defined here, called later)
11. `getCurrentHTML` / `cleanEditingArtifacts` / `downloadHTML` / `saveCurrent` / `saveAs`
12. View modes: `setView`, `updateIframePreview`, source-position helpers
13. Toolbar wireup
14. Source view sync helpers
15. File loading (`detectEncoding`, `decodeBytes`, `readFile`)
16. Open File / Paste / New button wireup
17. Drop zone
18. Save / Save As / Source toggle
19. Keyboard shortcuts
20. Image editing (handle, toolbar, drag, alignment)
21. Element Styles panel (refs, helpers, palette, bg image)
22. Patch into `setupImageEditing` to also setup panel listeners

Any new `let` you add must come before its first read. If you get `ReferenceError: Cannot access 'X' before initialization`, you've hit a TDZ bug.

## File loading & encoding detection

`readFile(file)` reads the file as an ArrayBuffer (not text), then:

1. `detectEncoding(bytes)` — checks BOM markers (`EF BB BF`, `FF FE`, `FE FF`) and scans the first 4KB for a `<meta charset>` declaration via ASCII decode. Returns the declared encoding or `null`.
2. `decodeBytes(bytes)` — if encoding declared, uses TextDecoder with that label. Else tries strict UTF-8 (`{fatal: true}`); if that throws (illegal byte sequence), falls back to Windows-1252.
3. The resulting string goes into `loadHTML(text, file.name)`.

Handles every HighLevel export I've seen (UTF-8, Windows-1252, ASCII) and most Word/Outlook output (Windows-1252).

A toast is shown when a non-UTF-8 encoding was used so the user knows the file will be saved as UTF-8 going forward.

## Fragment vs full page

`isFullPage(html)` — regex test for `<html>` or `<!doctype html>`. Used to decide whether to wrap.

`wrapFragment(fragment)` — returns a `<!DOCTYPE html>` wrapper with default body styles for preview. Reads `widePreview` to set max-width and padding. The wrap is visual-only — never persists to saved output.

`getCurrentHTML()` — for fragments, clones `doc.body` and returns its `innerHTML`. For full pages, clones `documentElement` and returns `<!DOCTYPE html>\n<html>...`. Either way, calls `cleanEditingArtifacts(clone)` first.

Saved/copied output never includes the wrapper. The synthetic `<style>` block is in the wrapped iframe DOM but `cleanEditingArtifacts` removes the editor-injected styles (`#cw-img-styles`, `#cw-sel-styles`); the wrapper itself isn't included for fragments because we only return body's `innerHTML`.

## The iframe lifecycle

`loadHTML(html, filename)`:

1. Sets `state.isFragment`, `state.filename`, `state.fileHandle = null`.
2. Sets `body.is-fragment` class.
3. Wraps fragment if needed; assigns to `frame.srcdoc`.
4. Hides the welcome screen.
5. Sets `frame.onload` to a function that runs once the new doc is ready:
   - `doc.designMode = 'on'`
   - Adds `selectionchange`, `keyup`, `mouseup` listeners (for `updateToolbarState`)
   - Calls `setupImageEditing(doc)` (now wrapped to also setup panel listeners)
   - Calls `setupIframeToSourceSync(doc)` for split-mode two-way sync
   - Calls `rebuildSourcePositions()` for click-to-locate
   - Focuses the body

Re-loading via `frame.srcdoc =` resets the iframe's document entirely. All the listeners are reattached on each load.

`updateIframePreview(html)` is a separate path used **only in split mode** to avoid the flicker of a full reload. It uses `doc.open() / doc.write() / doc.close()` to rewrite the existing document in place. This still clears event listeners on the doc, so the function reattaches them all.

## View modes

Three body classes: `view-page`, `view-source`, `view-split`. CSS targets these to show/hide the iframe and CodeMirror.

```css
body.view-page #editor-frame { display: block; }
body.view-page .CodeMirror { display: none; }
body.view-source #editor-frame { display: none; }
body.view-source .CodeMirror { display: block; }
body.view-split #editor-frame { display: block; flex: 1 1 50%; min-width: 0; }
body.view-split .CodeMirror { display: block; flex: 1 1 50%; min-width: 0; border-right: 1px solid var(--border); }
```

`setView(mode)` handles transitions:

- **page → source/split**: pull current iframe HTML into CodeMirror via `setSourceText(getCurrentHTML())`
- **source/split → page**: reload iframe with current source via `loadHTML(getSourceText(), ...)` — re-enables designMode
- **source → split**: push source to iframe preview via `updateIframePreview`
- **split → source**: no sync needed (iframe is already in sync)

Wrap toggle and Wide toggle visibility is CSS-driven on `body.view-source` / `body.is-fragment`.

The view choice is persisted in localStorage but doesn't auto-restore on load (initial view is always Page so the welcome screen makes sense).

## CodeMirror integration

CDN URL pattern: `https://cdnjs.cloudflare.com/ajax/libs/codemirror/5.65.16/...`

Loaded:

- `codemirror.min.css` + `codemirror.min.js`
- `theme/dracula.min.css`
- `mode/xml/`, `mode/javascript/`, `mode/css/`, `mode/htmlmixed/`
- `addon/edit/closetag.min.js`, `addon/edit/matchbrackets.min.js`
- `addon/search/search.min.js`, `addon/search/searchcursor.min.js`, `addon/dialog/dialog.min.js` + `addon/dialog/dialog.min.css`

Init:

```js
cm = CodeMirror.fromTextArea(sourceView, {
  mode: 'htmlmixed',
  theme: 'dracula',
  lineNumbers: true,
  lineWrapping: wrapEnabled,
  indentUnit: 2, tabSize: 2,
  autoCloseTags: true,
  matchBrackets: true,
  extraKeys: {
    'Ctrl-F': 'findPersistent', 'Cmd-F': 'findPersistent',
    'Ctrl-H': 'replace', 'Cmd-Alt-F': 'replace'
  }
});
```

Critical CSS gotcha: the `.CodeMirror` wrapper must NOT have `display: flex` set on it directly. CodeMirror uses absolute positioning for its internal pieces; making the wrapper a flex container makes them all row-flex items and the layout breaks. Use `display: block` (and `flex: 1` is fine because that's about how it sizes inside a flex parent — `<main>` is a flex container).

`cm.refresh()` must be called after the source view becomes visible. CodeMirror initializes its dimensions based on the visible size; if the textarea is hidden when `fromTextArea` runs, the editor renders with zero height. The transition functions call `cm.refresh()` in a `setTimeout(..., 0)` to ensure the CSS update has applied first.

## Two-way sync (split mode)

Two debounced flows:

**Source → Iframe:**

```js
cm.on('change', (cmInst, change) => {
  if (change.origin === 'setValue') return;
  rebuildSourcePositions();
  if (state.view !== 'split') return;
  clearTimeout(splitSyncTimer);
  splitSyncTimer = setTimeout(() => updateIframePreview(getSourceText()), 500);
});
```

The `change.origin === 'setValue'` filter is essential — it stops the feedback loop when the iframe-to-source path also calls `cm.setValue()`.

**Iframe → Source:**

```js
function setupIframeToSourceSync(doc) {
  let timer;
  doc.addEventListener('input', () => {
    if (state.view !== 'split') return;
    if (cm && cm.hasFocus()) return; // user typing in source — don't disturb
    clearTimeout(timer);
    timer = setTimeout(() => {
      const cursor = cm.getCursor();
      const scroll = cm.getScrollInfo();
      cm.setValue(getCurrentHTML());
      try { cm.setCursor(cursor); } catch(e) {}
      cm.scrollTo(scroll.left, scroll.top);
      rebuildSourcePositions();
    }, 600);
  });
}
```

The `cm.hasFocus()` guard prevents the iframe from yanking the source pane while you're typing in it. Cursor and scroll position are captured-and-restored to make the experience less jarring (still imperfect — a major rewrite resets cursor to start).

## Click-to-locate

`buildElementPositions(source, isFragment)` walks the source string with regex:

1. If fragment, start at index 0. If full page, start after `<body>` opening tag.
2. Skip comments (`<!--` ... `-->`) entirely.
3. For `<script>` and `<style>` — record the opening tag, then jump past the closing tag (skipping content).
4. For other opening tags — record `{tag: 'div', line: N}`.
5. Returns an array indexed by element document order.

`elementIndexInBody(el, doc)` walks `doc.body.querySelectorAll('*')`, skipping `data-cw-handle` elements (our injected resize handle), and returns the index of `el`.

The two indexes line up: the Nth opening tag in source = the Nth element in body's tree walk. Click → look up `sourcePositions[N].line` → `cm.scrollIntoView` + `cm.addLineClass`.

The mapping is approximate. Edge cases that break it:

- Heavily formatted source with awkward whitespace or unconventional comment placement
- Self-closing void tags counted differently than the regex thinks

If the highlight lands on the wrong line, the user can still click the breadcrumb to navigate.

## Image editing

`selectImage(img)`:

1. Calls `deselectImage()` to clear any previous selection.
2. Adds `cw-selected-img` class (gives the blue outline via injected style).
3. Populates the W input with current width.
4. Updates align button active state by reading `currentAlign(img)`.
5. Shows the image toolbar.
6. Calls `positionImageToolbar()` — calculates position based on `frame.getBoundingClientRect()` + `img.getBoundingClientRect()`.
7. Calls `addResizeHandle(img)` — creates a `data-cw-handle` div in the iframe doc.

The resize handle is in the iframe doc (so it scrolls with the content). Drag flow: mousedown captures starting X and width, mousemove updates `img.style.width` and the handle position, mouseup clears the dragging flag.

`currentAlign(img)` reads `img.style.float`, `img.style.display`, and `img.style.marginLeft` to determine which align button is active. `applyAlign(img, align)` sets the corresponding inline styles.

`cleanEditingArtifacts(root)` strips:

- All `[data-cw-handle]` elements (resize handle)
- `#cw-img-styles` (the injected outline-style)
- `#cw-sel-styles` (the dashed selection-element outline)
- `cw-selected-img` and `cw-selected-element` classes

It also normalizes the meta charset to UTF-8.

## Element Styles panel

Toggled via `body.panel-open`. Persisted in localStorage as `cw-editor-panel`.

`setSelectedElement(el)`:

1. Removes `cw-selected-element` from the previous `selectedEl`.
2. Sets `selectedEl = el`.
3. Adds the class on the new element (gives the dashed blue outline).
4. Calls `refreshPropsPanel()`.

`refreshPropsPanel()`:

1. Builds the breadcrumb (clickable spans with the ancestor chain, root → leaf, current marked with `.current`).
2. Reads computed `backgroundColor`, `color`, `opacity` from `getComputedStyle(selectedEl)`.
3. Sets the color pickers and hex inputs.
4. Calls `refreshBgImagePanel()`.

The panel listeners are added by `setupPanelListeners(doc)` in capture phase, so they run alongside (not inside) the image editing click handler. They:

- Update `selectedEl` on iframe click
- Capture the iframe text selection on click/mouseup/keyup
- Toggle `body.has-text-selection` based on whether the cached range is non-collapsed

## Selection-aware text/bg color

The trick: when a user selects "Leadership" inside an h1 and then clicks a panel control, the iframe loses focus and the visual selection highlight disappears — but the selection range itself is still alive in the iframe's selection object.

`captureIframeSelection()` runs on iframe click/mouseup/keyup and saves `lastIframeRange = sel.getRangeAt(0).cloneRange()`.

When the user clicks a color, `applyFg` / `applyBg`:

1. Check `hasTextSelection()` — true if `lastIframeRange && !collapsed`.
2. If true, call `restoreIframeSelection()` — `doc.body.focus()`, `sel.removeAllRanges()`, `sel.addRange(lastIframeRange)`. Then `doc.execCommand('styleWithCSS', true)` followed by `execCommand('foreColor' / 'hiliteColor', false, color)` — the selection gets wrapped in a styled `<span>`.
3. If false, write `selectedEl.style.color = color` (or `backgroundColor`).

`styleWithCSS = true` makes execCommand emit `<span style="color: ...">` instead of legacy `<font color="...">`. Cleaner output.

The hint spans (`.sel-hint` / `.el-hint`) toggle visibility based on `body.has-text-selection`, giving the user visual feedback about which mode the next color change will use.

## Background image inspector

`parseBgImageUrl(bgImage)` — regex extract from `url("...")` or `url(...)`. Handles only the first `url(...)` for layered backgrounds.

`refreshBgImagePanel()`:

1. Reads computed `backgroundImage`, `backgroundSize`, `backgroundPosition`, `backgroundRepeat`, `backgroundAttachment`.
2. If has image, populates URL field, preview thumbnail, and dropdowns.
3. Toggles `bg-img-section.has-image` to switch between empty state and editing state.

`normalizeBgPosition(v)` maps computed values like `'50% 50%'` to keyword form `'center center'` so the dropdown can match.

Setting writes inline styles. Clearing removes all five bg-image-related properties at once.

## Page palette

`buildPagePalette()` walks two sources:

1. **Computed styles**: `doc.body.querySelectorAll('*')` → for each, read `color`, `backgroundColor`, four border colors. Filter out transparent and empty. Convert to hex via `rgbToHex()`. De-duplicate via Set.

2. **CSS custom properties**: `Array.from(doc.styleSheets)` → for each rule, iterate `rule.style` looking for properties starting with `--`. Add their values to the set if they look like color literals.

Each unique color becomes a `.palette-swatch` button:

- Click: copies hex via `navigator.clipboard.writeText(hex)`.
- Hover reveals two micro-buttons (`apply-buttons`): "BG" and "Txt" call `applyBg(hex)` / `applyFg(hex)`.

The hover-revealed buttons go inside the swatch as absolutely-positioned children.

## Wide preview

`widePreview: bool`, persisted as `cw-editor-wide` in localStorage.

`wrapFragment(fragment)` reads it: `max-width: ${widePreview ? 'none' : '880px'}` and `padding: ${widePreview ? '0' : '24px'}`.

The button only shows when `body.is-fragment`. Toggling rewraps:

- In split mode → `updateIframePreview(getSourceText())`
- Otherwise → `loadHTML(getCurrentHTML(), state.filename)`

## Save / Save As

`downloadHTML(html?)` — Blob + invisible `<a download>` click. Used as the fallback path. Optional `html` arg lets `saveAs` pass a pre-computed string.

`saveCurrent()`:

1. If `state.fileHandle`, attempt direct overwrite via `handle.createWritable()` → `writable.write(html)` → `writable.close()`. May need `requestPermission({mode:'readwrite'})` on first call.
2. Otherwise, `downloadHTML(html)`.

`saveAs()`:

1. If `window.showSaveFilePicker`, call it with `suggestedName` and HTML accept type. On success, write the file and store the handle.
2. Otherwise, `prompt()` for a filename and call `downloadHTML(html)`.

`AbortError` from the picker = user canceled the dialog (do nothing, no toast).

`state.fileHandle` is cleared in `loadHTML` so opening a different file resets the link.

## Cleanup pipeline (export)

Called by `getCurrentHTML()` (which is called by Save, Save As, Copy, and source-mode entry).

`cleanEditingArtifacts(root)`:

1. Removes `[data-cw-handle]` elements (image resize handle).
2. Removes `#cw-img-styles` (image selection outline injected style).
3. Removes `#cw-sel-styles` (element selection outline injected style).
4. Removes `cw-selected-img` and `cw-selected-element` classes.
5. Normalizes meta charset to a single `<meta charset="UTF-8">` at the start of head.

## localStorage keys

| Key | Values | What |
|---|---|---|
| `cw-editor-wrap` | `'1'` / `'0'` | CodeMirror line wrapping |
| `cw-editor-wide` | `'1'` / `'0'` | Wide preview for fragments |
| `cw-editor-panel` | `'1'` / `'0'` | Element Styles panel open |

`cw-editor-view` is reserved but not currently used — view always starts at Page on page load.

## Common gotchas

1. **`let` declarations in TDZ.** If you add new state vars, put them above their first read. The IIFE runs top-to-bottom; click handlers don't run yet but the line that reads `widePreview` to set initial button state does. See "Initialization order" above.

2. **CodeMirror `display: flex` breaks layout.** Use `display: block`. The internal pieces use absolute positioning that flex items mess up.

3. **`doc.open() / write() / close()` clears all event listeners on the document.** Reattach in `updateIframePreview` and `frame.onload`.

4. **Function declarations are hoisted; `const` arrow functions are not.** Putting helpers below the call site works for `function name()` but not `const name = () => {}`. The codebase mostly uses `function` declarations for this reason.

5. **execCommand `styleWithCSS` is per-document and resets on doc rewrite.** Call it before each `foreColor` / `hiliteColor` to ensure inline-style output.

6. **File System Access permission can prompt mid-save.** The Chrome flow is: first `createWritable()` may need explicit permission. We handle this in `saveCurrent` via `queryPermission` + `requestPermission`.

7. **Iframe selection doesn't fire `selectionchange` on the parent doc.** That's why we listen for it on the iframe doc specifically. Same for `input` and `keyup`.

8. **CodeMirror's `change` event fires for `setValue` too.** Filter by `change.origin === 'setValue'` to avoid feedback loops.

9. **`getComputedStyle(el).backgroundColor` returns `'rgba(0, 0, 0, 0)'` for transparent**, not the empty string. Filter accordingly in palette extraction and in panel value reads.

10. **Setting `srcdoc` triggers `frame.onload` async.** Code that depends on the doc being ready must go inside the onload callback (or use `await new Promise(r => frame.onload = r)`).

## Where to add new features

| Feature kind | Where it goes |
|---|---|
| New header button | `<header><div class="header-actions">` + JS wireup near other button handlers |
| New formatting toolbar button | `<div id="toolbar">` (use `data-cmd` for execCommand-style commands; or new id for custom) |
| New panel section | Inside `#props-panel .panel-body` + matching JS in the panel block |
| New keyboard shortcut | `document.addEventListener('keydown', ...)` block |
| New iframe behavior | Inside `frame.onload` callback in `loadHTML`, AND inside `updateIframePreview`, AND inside `setupImageEditing` (depending on lifecycle) |
| New image-toolbar control | `<div id="image-toolbar">` HTML + JS handler near the existing `img-*` handlers |
| New cleanup rule | `cleanEditingArtifacts(root)` — runs on every export |
| New localStorage key | Use the `cw-editor-*` prefix convention |

## Future feature ideas

- **Insert table** with rows/columns picker; contextual cell controls (insert row/column, merge, set border)
- **Multi-layer background image** support (parse comma-separated `background-image`)
- **Word-level text color** as a top-toolbar button (distinct from the panel; saves a step for inline highlights)
- **File handle persistence via IndexedDB** so Save remembers the file across browser sessions (current behavior: handle dies on tab close)
- **Better cursor preservation** in split mode (current implementation captures index, may drift on big edits)
- **CSS variable inspector** in the panel (read/edit `--var-name` values declared on the selected element or its ancestors)
- **Padding / margin / border controls** in the panel
- **Responsive preview** (resize handle on the editor frame to test mobile/tablet widths)
- **Built-in link checker** — flag broken `<a href>` and `<img src>` URLs
- **Undo for inline-style edits** beyond what designMode tracks (the panel changes don't go through execCommand, so they're outside undo history)

## Testing checklist for new features

After any non-trivial change:

1. Open the file in Chrome. Welcome screen appears.
2. Click "Start Blank" — type, format, save (Ctrl+S downloads).
3. Click "Open File" — pick block.html. Page mode renders the design correctly.
4. Switch to Source — code shows with line numbers and dracula theme.
5. Switch to Split — both panes show; type in source, preview updates after 500ms.
6. Click an element in split — source line highlights.
7. Type in preview — source updates after 600ms.
8. Click the Styles button — panel opens with breadcrumb.
9. Select a word → click a palette swatch's "Txt" button → just that word changes color.
10. Click an image → resize handle and toolbar work; URL/Alt prompts work; alignment buttons work.
11. Save As → native dialog appears in Chrome; pick a location.
12. Edit, Ctrl+S → no dialog, file overwritten directly.
13. Open File again — fileHandle resets; next Save downloads.
14. Toggle Wide → fragment renders edge-to-edge.
15. Copy HTML → check clipboard for clean output (no `data-cw-handle`, no `cw-selected-*` classes).

If any step fails, check the browser console for thrown errors. The IIFE doesn't catch errors — they bubble to the console, where they're easy to spot.

## Version history

- **v1.0** — Initial release. Visual editor + simple textarea source view. Header buttons (Open, Paste, New, Source, Copy, Save). Image editing toolbar.
- **v1.1** — Encoding detection (Windows-1252 fallback). Fixed truncated-script bug. Auto-charset normalization on export.
- **v1.2** — CodeMirror source view with line numbers + Dracula theme + find/replace + wrap toggle. Three view modes (Page/Source/Split) with two-way live sync and click-to-locate. Element Styles panel with breadcrumb, BG/text colors, opacity, background image inspector, auto-extracted page palette. Selection-aware color application (foreColor/hiliteColor on text selections, inline style on elements). Image URL/Alt buttons. Wide preview toggle for fragments. Save / Save As with File System Access API. Officially named Camille.
