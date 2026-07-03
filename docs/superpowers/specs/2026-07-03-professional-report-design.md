# Professional Survey Report Upgrade — Design

Date: 2026-07-03
Branch: `professional-report`
Scope: `index.html` only (single-file app). Do not touch `mobile.html`, `brinc_tank_gps_professional.html`, or the untracked zip packages.

## Purpose

The app produces a Word site-survey report, but it does not look like an engineering document and carries almost no customer information. The only customer field is a single "Agency name" input used for file names and one table row. This upgrade adds a full project-information workflow and rebuilds the Word export into a professional engineering-style document, while keeping the single-file, in-browser architecture.

## 1. Project Info panel (UI)

A collapsible "Project Info" panel near the top of the app replaces the lone agency-name input.

**Customer block**
- Agency name (existing value migrates here)
- Point of contact: name, title, phone, email
- Site address
- Project / job number

**BRINC block**
- Installer name, title, phone, email
- Persists in `localStorage` across sessions (survives new surveys)

**Document control (light)**
- Revision — text input, default "A"
- Status — dropdown: Draft / For Review / Issued (default Draft)
- Survey date — auto-filled from photo EXIF (existing `formatSurveyDateDisplay` logic), user-overridable

**Behavior**
- Panel collapses after fill; collapsed state shows a one-line summary, e.g. "Mesa PD — Job #2026-041 — Draft Rev A".
- "New project" button clears the customer block and document-control fields, keeps the BRINC block.
- All fields saved to `localStorage`; customer fields cleared only via "New project".
- Empty fields are allowed; export renders them as "—".

## 2. Photo captions

- Caption text input added to the existing annotation modal.
- Caption stored on the photo object, in-memory like all other annotations (the app has no annotation persistence; captions follow the same model).
- Captions appear in the ZIP export's per-location `Location_Summary.txt` alongside existing photo metadata.
- Word figure line becomes "Figure N. <caption>". Empty caption falls back to "Figure N — <location name>". Raw file names no longer appear as figure captions.

## 3. Word document rebuild (`buildWordDocumentBlob`)

**Cover page**
- BRINC logo, base64-embedded in `index.html` (source: `brinc_tank_gps_production_package/assets/brinc-logo.png`).
- Report title + subtitle.
- Customer info table (all customer-block fields).
- Prepared-by table (BRINC block).
- Revision / status / survey date line.
- Page break after cover.

**Table of contents**
- Static linked list: one entry per location section, internal bookmark hyperlink, no page numbers.
- Chosen over a real TOC field because Word on iOS cannot update fields — a TOC field renders blank on iPhone (Word iOS, Quick Look, Mail preview) until opened in desktop Word. Static bookmark links work everywhere.

**Header / footer**
- Header on every page after the cover: agency + job number on the left, BRINC on the right.
- Footer: page number, document status, revision.

**Body**
- Each location section gets a bookmark anchor matching its TOC entry.
- Continuous figure numbering across sections (existing behavior, now with captions).
- Existing per-photo metadata (date, coordinates, markers, symbols, Google Maps links) unchanged.

## 4. Error handling

- Empty project fields render "—", never blank cells.
- Logo missing/undecodable → cover renders without logo, no crash.
- Existing export guard unchanged (requires at least one annotated photo).
- `docx` library load failure message unchanged.

## 5. Testing (manual)

- Export with all fields filled and with all fields empty; verify "—" fallbacks.
- Open generated .docx in desktop Word and on iPhone (Word iOS + Quick Look): cover, TOC links jump to sections, header/footer correct on both.
- Captions: set captions, export ZIP, verify captions in `Location_Summary.txt`; verify Word figure lines use captions with location-name fallback.
- `localStorage`: reload page, BRINC block persists; "New project" clears customer fields only.

## Out of scope

- PDF export, Word templates (docxtemplater), signature blocks / full revision-history table.
- Any changes to photo grouping, annotation tools, or ZIP image export.
