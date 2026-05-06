<img src="https://assets.cdn.filesafe.space/e3LOYAHOBn0PFjgBHsPS/media/69fbcdf4533d641a893d0934.webp" alt="Camille — Visual Composition for the AI-Built Web" width="600">

# Camille v1.2 — User Guide

*Visual Composition for the AI-Built Web*

A self-contained WYSIWYG HTML editor that runs in your browser. Single file, no installation, no server. Designed for editing content blocks (the kind you paste into HighLevel's Custom HTML element) and standalone HTML pages alike.

> Live at **[camille.sypmedia.com](https://camille.sypmedia.com)** · Copyright © 2026 SYP Media LLC. All rights reserved.

## Quick start

Visit **[camille.sypmedia.com](https://camille.sypmedia.com)** in your browser, or open the local `index.html` file directly. Chrome and Edge have a few extra capabilities — see [Browser support](#browser-support) at the end.

The welcome screen offers four ways to get started:

- **Open HTML File** — pick a `.html` file from your computer
- **Paste HTML** — paste markup into a textbox (works for both fragments and full pages)
- **Start Blank** — begin with an empty document
- **Drag and drop** an HTML file onto the editor

Once content is loaded, the welcome screen disappears and the toolbar comes alive.

---

## The three view modes

A segmented control near the top right (**Page · Source · Split**) lets you switch between three ways of looking at and editing your document.

### Page view

Pure visual editing. Click anywhere in the document and start typing. Highlight text and use the formatting toolbar above (Bold, Italic, headings, lists, links, etc.). Click an image to bring up its contextual toolbar. Click any other element to update the Element Styles panel.

### Source view

Pure code editing. The whole pane shows your raw HTML in a dark Dracula theme with line numbers, syntax highlighting, auto-close tags, and bracket matching.

- **Ctrl/Cmd+F** — find (with a regex toggle)
- **Ctrl+H** or **Cmd+Alt+F** — find and replace
- A **Wrap** button appears in the header when source is showing — toggle whether long lines wrap visually (your file content is unchanged either way)

### Split view

Source on the left, live preview on the right — fully editable on both sides. Type in the source pane and the preview updates 500ms after you stop. Edit visually in the preview and the source updates 600ms after you stop. Whichever pane has focus is the "driver" — the other side won't override your cursor while you're working.

**Click-to-locate**: clicking any element in the preview pane scrolls the source pane to the corresponding line and highlights it. Useful for finding "where is *this* in the code?"

---

## Header buttons

Left to right across the header:

| Button | What it does |
|---|---|
| **Open File** | File picker for a local HTML file |
| **Paste HTML** | Modal for pasting markup |
| **New** | Start with a blank document |
| **Page / Source / Split** | View mode (segmented control) |
| **Wrap** | Toggle source line wrapping (visible only when source is showing) |
| **Wide** | Toggle wide preview for fragments (visible only for fragments) |
| **Styles** | Show/hide Element Styles panel |
| **Copy HTML** | Copy current document to clipboard |
| **Save** | Save to disk (overwrite linked file, or download) |
| **Save As** | Save to a new file (native dialog or filename prompt) |

---

## Formatting toolbar (Page mode)

The second row in Page and Split modes has standard text-editor controls:

- **Block format dropdown** — Paragraph, Heading 1–4, Quote, Code Block. Applies to whichever paragraph the cursor is in.
- **B / I / U / S** — Bold, Italic, Underline, Strikethrough on selected text. Keyboard: Ctrl+B, Ctrl+I, Ctrl+U.
- **• List / 1. List** — bulleted and numbered lists for selected paragraphs.
- **← / →** — outdent / indent (decrease and increase list level).
- **Align icons** — left, center, right alignment for the current paragraph.
- **Link** — prompt for a URL; if text is selected, wraps it in `<a href>`. If you have an existing link selected, the prompt pre-fills with its current URL.
- **Unlink** — remove the link wrapping the current selection.
- **Clear** — strip formatting from the selection.
- **Undo / Redo** — Ctrl+Z / Ctrl+Shift+Z.

---

## Editing images

Click any image in Page or Split mode. A floating toolbar appears just above it with these controls:

- **W: [number] px** — change the image width in pixels. Aspect ratio is preserved automatically.
- **Resize handle** — a blue circle at the bottom-right corner of the image. Drag to resize.
- **Left / Center / Right / Inline** — alignment / float behavior:
  - *Left*: floats left, text wraps to the right
  - *Right*: floats right, text wraps to the left
  - *Center*: centers on its own line
  - *Inline*: no float, flows with surrounding text
- **URL** — prompt to view or change the image source URL
- **Alt** — prompt to view or change the alt text (for screen readers and SEO)
- **Delete** — remove the image

Click anywhere outside the image, or press **Esc**, to deselect.

The selection outline and resize handle are stripped on Save / Copy — they never end up in your exported HTML.

---

## Element Styles panel

Click the **Styles** button to open a 300-pixel panel on the right. Click any element in the editor and the panel updates with controls for that element.

### Selected element breadcrumb

Shows the full ancestor chain (e.g. `body › section.dtd-hero › div.dtd-container › h1`). Each segment is clickable — click an ancestor to "select up the tree." Useful when you've clicked a child but want to style its container.

### Background

A native color picker, a hex text field, and a × clear button. Behavior depends on whether you have text selected:

- **No text selection** → applies as inline `background-color` on the selected element
- **Text selected** → highlights just that text via `execCommand('hiliteColor')` (like a marker)

### Text color

Same three controls. With no text selection, applies to the whole element. With text selected, applies to just the selected text via `execCommand('foreColor')` — a `<span style="color: ...">` wraps the selection.

The section labels show a hint (`→ entire element` vs `→ selected text only`) so you always know which behavior is active.

### Opacity

Slider from 0–100%. Applies as inline `opacity` on the selected element.

### Background image

Auto-detects whether the selected element has a background image.

If not, shows a "+ Add background image" button. Click it, paste a URL, and the editor sets sensible defaults (cover size, center position, no-repeat).

If yes, shows:

- A **preview thumbnail** of the image
- An **editable URL field** with × to clear (clearing also removes related size/position/repeat/attachment styles)
- **Size** — auto / cover / contain / stretch (`100% 100%`)
- **Position** — center, top, bottom, four corners
- **Repeat** — repeat / no-repeat / repeat-x / repeat-y
- **Attachment** — scroll / **fixed (parallax)** / local

Setting Attachment to *fixed* gives a parallax effect — the background stays still while content scrolls over it.

### Page palette

Auto-extracted color swatches from the page. Two sources feed it:

- Every visible element's computed `color`, `background-color`, and border colors
- Every CSS custom property in the page's stylesheets (e.g. `--dtd-blue: #005FAF`)

Each swatch supports three actions:

- **Click** the swatch — copies its hex to your clipboard
- **Hover** the swatch — two micro-buttons appear at the bottom: **BG** applies it as the selected element's background, **Txt** applies it as text color
- **Rescan** button (top-right of the section) — re-extract colors after major edits

---

## Saving your work

Camille has two save modes:

### Save (Ctrl+S)

If you've previously used **Save As** to point at a real file, **Save** overwrites that file directly — no dialog, no Downloads folder. Like a desktop app.

If you haven't, **Save** falls back to downloading a copy to your browser's Downloads folder, using the current filename.

### Save As (Ctrl+Shift+S)

In Chrome / Edge / Brave / Opera / Arc, opens the native Save dialog where you pick the folder and filename. After saving, the editor remembers that file — future Saves overwrite it directly.

In Firefox / Safari, falls back to a filename prompt followed by a download (browser settings determine the location).

### When the file link breaks

Opening a different file via **Open File** clears the remembered file handle. This prevents accidentally overwriting File A when you've moved on to editing File B. After opening the new file, use **Save As** again to link it.

### Browser may ask for permission

The first time you Save after a Save As, Chrome may show "Allow this site to edit the file?" — say yes once and it'll remember for the session.

---

## Common workflows

### Editing a HighLevel content block

1. In HighLevel, open the Custom HTML element and copy its HTML.
2. In Camille, click **Paste HTML**, paste, click **Load**.
3. Edit visually in Page mode (or in Source mode for fine control).
4. Click **Copy HTML** to get the cleaned-up output.
5. Paste back into HighLevel's Custom HTML field. Save in HighLevel.

The Wide toggle is handy here — content blocks are typically full-width in HighLevel but Camille shows them in a 880-pixel reading column by default. Toggle Wide to preview them edge-to-edge.

### Editing a local HTML file

1. **Open File** → pick the file. Encoding is auto-detected.
2. Edit.
3. **Save As** the first time — pick the location and filename (overwrite the original or save a copy).
4. **Ctrl+S** for subsequent saves — overwrites the linked file directly.

---

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| Ctrl+S | Save |
| Ctrl+Shift+S | Save As |
| Ctrl+B / Ctrl+I / Ctrl+U | Bold / Italic / Underline (Page mode) |
| Ctrl+Z / Ctrl+Shift+Z | Undo / Redo |
| Ctrl+F | Find (Source mode) |
| Ctrl+H | Find & Replace (Source mode) |
| Esc | Close paste modal / deselect image |

(On Mac, use Cmd in place of Ctrl.)

---

## File compatibility

### Encoding

Camille auto-detects file encoding when you open:

- **BOM markers** (UTF-8, UTF-16 LE/BE) — used directly
- **`<meta charset>` declaration** in the file — read and used
- Otherwise, tries strict UTF-8. If that fails, falls back to **Windows-1252** (covers most legacy HTML, Word/Outlook exports, older HighLevel files)

A toast confirms when a non-UTF-8 encoding was detected. **All files are saved as UTF-8** with a `<meta charset="UTF-8">` written into the head, so re-saving an old Windows-1252 file modernizes it.

If you see a `�` (replacement) character on a published page, the original byte was already destroyed before Camille got it. Re-load the original source file with Camille's auto-detection and the smart punctuation will come back. If you don't have an unmodified original, edit the `�`s manually in source view.

### Fragments vs full pages

Camille distinguishes between:

- **Full pages** — files that contain `<html>`, `<head>`, and `<body>`. Rendered as-is.
- **Fragments** — content blocks like `<div class="hero">...</div>`. Wrapped in a synthetic page with editor-friendly default styles for preview only — never written to your saved file.

The distinction is automatic. The Wide toggle only affects fragments.

---

## Limitations

- The **Background image inspector** edits only the first image of a layered (comma-separated) `background-image`. Multi-layer designs need source view.
- **Tables**: no Insert Table UI yet. Pasted/loaded tables render and can be edited cell-by-cell, but there's no rows/columns picker.
- **Email-specific HTML** (mso-* properties, conditional comments) renders best-effort but the editor isn't an Outlook simulator.
- **Element selection in Split mode** can land on a styling `<span>` if you previously colored some text — use the breadcrumb to navigate up to the parent element.

---

## Browser support

| Feature | Chrome/Edge/Brave/Arc | Firefox | Safari |
|---|---|---|---|
| Core editing | yes | yes | yes |
| Source view (CodeMirror) | yes | yes | yes |
| Native Save dialog | yes | fallback to prompt+download | fallback to prompt+download |
| Auto-close tags | yes | yes | yes |
| Find & replace | yes | yes | yes |

The "fallback" cells use a `prompt()` for filename and trigger a download — works fine, just less app-like.

---

## What's coming next

Possible features for v1.3+:

- Insert Table button with rows/columns picker
- Multi-layer background image editor
- Word-level text color in the formatting toolbar (as a top-toolbar button distinct from the Styles panel)
- Persistent file handles (so Save remembers the file across browser sessions)
- Padding / margin / border controls in the panel
- Responsive preview (drag handle to test mobile/tablet widths)
