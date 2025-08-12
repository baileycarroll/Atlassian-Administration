---

```
title: Import Notes (Obsidian ↔ Confluence)
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Docs/Platform]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, docs, obsidian, confluence, import]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial guidance for moving Markdown from Obsidian to Confluence with minimal cleanup. *Ref: `[Change/Ticket ID]`*

# Purpose

Offer a practical, low‑friction path to publish this guide from an **Obsidian vault** into **Confluence** (Cloud or Data Center). Focus on **portable Markdown**, predictable anchors, and easy post‑import fixes.

---

# Supported Markdown & Key Differences

| Area        | Obsidian                      | Confluence Behavior                                              | Recommendation                                            |
| ----------- | ----------------------------- | ---------------------------------------------------------------- | --------------------------------------------------------- |
| Headings    | `#`..`######`                 | Imported as page headings (H1 becomes page title or top heading) | Use H1–H3 primarily for a clean page outline              |
| Bold/Italic | `**bold**`, `*italic*`        | OK                                                               | Standard Markdown                                         |
| Links       | `[Text](relative/page.md)`    | May import as plain links; won’t resolve until pages exist       | Keep relative links; fix to page links post‑import        |
| Images      | `![alt](images/file.png)`     | Becomes an attachment or external link                           | Attach images to the page or a parent; re‑point as needed |
| Tables      | GitHub‑style tables           | Imported as Confluence tables                                    | Keep tables simple; avoid nested tables                   |
| Code Fences | \`\`\`lang + content          | Imported as code blocks                                          | Always specify a language for syntax highlighting         |
| Callouts    | Obsidian callouts `> [!note]` | Not native                                                       | Convert to Confluence **Info/Warning** macros post‑import |
| HTML        | Inline HTML allowed           | Mixed                                                            | Avoid unless necessary                                    |

> This guide already sticks to **portable Markdown** and avoids Obsidian‑only features.

---

# TL;DR Formatting Rules (used across this guide)

* **Front‑matter**: Wrap YAML metadata in a **code block** at the top (we do this on every page).
* **Headings**: Use `#` for H1 page title once, then H2/H3 for structure.
* **Links**: Prefer `[Text](relative/path.md)`; fix to Confluence page links after import.
* **Images**: Store under `/images/...`; reference with standard Markdown; attach in Confluence after import.
* **No wikilinks**: Don’t use `[[Wiki Links]]` without a Markdown equivalent.

---

# Pre‑Import Checklist

* [ ] Filenames are **kebab‑case** and match page titles where possible.
* [ ] No Obsidian **wikilinks** remain; all are standard Markdown links.
* [ ] All images referenced via relative paths (e.g., `images/topology.png`).
* [ ] Code blocks specify a language: `bash, `sql, \`\`\`xml, etc.
* [ ] Front‑matter is inside a code block (already standard here).
* [ ] Large tables kept simple; no nested Markdown inside cells.

---

# Import Methods (pick one)

## 1) Paste Markdown → Confluence Editor

* Create page → paste Markdown → Confluence autoconverts.
* Good for small pages; fix links/images afterward.

## 2) Markdown Importer (App/Plugin)

* Some instances use an importer app to convert `.md` and upload images.
* Best for bulk imports; check your Confluence add‑ons.

## 3) HTML Markup Import

* Convert Markdown → HTML locally; **Insert Markup** in Confluence.
* Preserves more formatting; images still need attaching.

> Choose the simplest method supported by your Confluence. For bulk, prefer an importer.

---

# Headings, Anchors & TOC

* Confluence auto‑generates **anchors** from headings; avoid duplicate headings on a page.
* Add a **Table of Contents** macro near the top (post‑import) if desired.
* If you must link to a section: create the link after import using Confluence’s anchor picker.

---

# Links & Cross‑References

* Keep links **relative** in Markdown (`../04-installation-initial-configuration.md`).
* After import, edit each link to point to the **Confluence page** (search by title) rather than the raw `.md`.
* For external references, leave as full URLs.

---

# Images & Attachments

* Place images under a vault folder like `images/`.
* After import, **attach** images to the page (or a parent index page) and update the image references if the importer didn’t.
* Prefer PNG/SVG for diagrams; compress large files.
* For diagrams, consider converting to **draw\.io (diagrams.net)** and embedding via the macro post‑import.

---

# Code Blocks & Config Snippets

* Always include a language for syntax highlighting: `bash`, `sql`, `xml`, `ini`, `apache`, `yaml`, `json`.
* Keep lines under \~120 chars where possible for readability.
* Avoid secrets in examples; placeholders are bracketed (see §17 Variable Glossary).

---

# Tables

* Keep header row + `|---|---|` separator simple.
* Avoid inline HTML or complex nesting in cells.
* For very wide matrices, split into multiple tables or pages.

---

# Callouts & Macros

* Convert Obsidian callouts to Confluence macros **after import**:

  * `> [!note]` → **Info** macro
  * `> [!warning]` → **Warning** macro
  * `> [!tip]` → **Tip** macro
* Keep macro usage light for portability.

---

# Special Characters & Escaping

* Escape pipes `|` inside table cells with `\|`.
* Use fenced code blocks for any YAML or JSON to avoid conversion quirks.
* Avoid leading/trailing spaces in headings (affects anchors).

---

# Post‑Import QA (5–10 minutes per page)

* [ ] Page title matches H1; headings are properly nested.
* [ ] TOC macro (optional) shows expected sections.
* [ ] All links resolve to Confluence pages, not `.md` files.
* [ ] Images render; alt text present.
* [ ] Code blocks retain language and indentation.
* [ ] Labels/tags applied for search (e.g., `admin-guide`, `jira`).

---

# Known Pitfalls

* **Front‑matter** treated as YAML by some importers → we wrap it in a **code block** to preserve it.
* **Wikilinks** don’t convert → always provide a Markdown equivalent.
* **Large tables** with complex Markdown → formatting loss; keep simple.
* **Deep relative paths** for images → break on import; attach images to the page and relink.
* **Inline HTML** may be stripped → avoid unless necessary.

---

# Automation Hints

* Use a pre‑publish script to:

  1. Validate links (no `[[wikilink]]`),
  2. Ensure code blocks are fenced with languages,
  3. Copy images to a predictable folder for upload.
* Maintain a mapping file from `relative/path.md` → `Confluence Page Title` to speed link fixing.

---

# Variables Used on This Page

`[Change/Ticket ID]`, `[YYYY‑MM‑DD]`.

# Import Notes

* This page **is** the import note; no extra tips needed. After publishing, consider adding Confluence‑specific screenshots of the import flow your instance supports.
