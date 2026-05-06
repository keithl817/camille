<img src="https://assets.cdn.filesafe.space/e3LOYAHOBn0PFjgBHsPS/media/69fbcdf4533d641a893d0934.webp" alt="Camille — Visual Composition for the AI-Built Web" width="600">

# Camille

**Visual Composition for the AI-Built Web** — a self-contained WYSIWYG HTML editor that runs in your browser. Single file, no installation, no server, no build step.

Live at **[camille.sypmedia.com](https://camille.sypmedia.com)**.

## What it's for

Editing HTML content blocks visually — the kind of content you'd paste into a HighLevel Custom HTML element, an email template, a CMS rich-text field, or a standalone HTML page. Camille opens whatever you give it (full pages or fragments), lets you edit visually or in code, and gives you cleaned-up output back.

## Highlights

- Visual WYSIWYG editor with a formatting toolbar (headings, lists, links, alignment, indent)
- CodeMirror-powered source view with line numbers, Dracula syntax highlighting, find & replace, auto-close tags
- Three view modes: **Page**, **Source**, **Split** — Split has live two-way sync between code and preview, plus click-to-locate (click an element in the preview, the source pane jumps to its line)
- Element Styles inspector — clickable breadcrumb, native color pickers for background and text, opacity, full background-image controls (size, position, repeat, attachment for parallax), and an auto-extracted page color palette pulled from your stylesheet
- Image editing toolbar — drag-handle resize, alignment (left/center/right/inline), URL editor, alt-text editor
- Save / Save As with native file dialogs (Chromium browsers via the File System Access API), with download fallback for Firefox/Safari
- UTF-8 / Windows-1252 encoding auto-detection on file open
- Wide preview toggle for fragments

## Run locally

Just open `index.html` in a modern browser. No setup. The CodeMirror dependencies load from cdnjs.

## Documentation

- **[User Guide](Camille-User-Guide.md)** — features, workflows, keyboard shortcuts, browser support
- **[Technical Notes](Camille-Technical-Notes.md)** — architecture, where features live, gotchas, extension guide

## Browser support

| Feature | Chrome / Edge / Brave / Arc | Firefox | Safari |
|---|---|---|---|
| Core editing | yes | yes | yes |
| Source view | yes | yes | yes |
| Native Save dialog | yes | fallback (download) | fallback (download) |
| Find & replace | yes | yes | yes |

## Hosting

This repo is configured for GitHub Pages with the custom domain `camille.sypmedia.com` (see `CNAME`). Pushing to `main` deploys automatically.

## License & copyright

Copyright © 2026 SYP Media LLC. All rights reserved.

---

*Camille uses [CodeMirror 5](https://codemirror.net/5/) loaded from cdnjs for the source editor.*
