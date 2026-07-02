# Annotation Import: survey drawing tools → gps app

Date: 2026-07-02
Branch: 2x
Status: approved by user

## Goal

Import the photo-annotation drawing tools from `Monster/survey/index.html` into
`Monster/gps/index.html` (BRINC TANK station photo organizer): freehand pen,
text labels, and five engineering symbol stamps. Existing bubble-marker system
stays unchanged and remains the default tool.

## Source material (survey app)

- Pen tool: pointer-drag freehand polyline, color picker + width slider.
- Text tool: click places typed text.
- Symbol stamps drawn as canvas vector paths with a thick contrast halo and a
  caption label: power ⚡ (#e60000, "PWR"), ethernet 🔌 (#0057b8, "ETH"),
  rf 📶 (#ff7a00, "RF"), crane 🏗 (#ffd100, "CRANE"), unistrut ▯ (#9ea3a8, "STRUT").
- All coordinates normalized 0–1. Rendering is Canvas 2D, baked into exports.
- Functions to port: `drawSymbolIcon`, `drawStroke`, `contrastColor`,
  pen/text/symbol pointer handlers (survey index.html ~21464–21768).

## Approach (chosen: extend existing system)

The gps app already has a canvas annotation modal with `markers:[]` per photo,
normalized coords, and a single render choke point `drawAnnotationCanvas()`
reused by modal, ZIP export, and Word export. The import adds a parallel
`drawings:[]` array and one extra draw pass — existing markers untouched.

Rejected alternatives: wholesale modal replacement (destroys bubble markers);
second overlay canvas (complicates export baking for no benefit).

## Data model

Each photo object gains `drawings: []` (both photo factories, currently
index.html:885 and :890). Stroke objects use the survey format verbatim:

- `{ type: 'pen', color, width, points: [{x, y}, ...] }`
- `{ type: 'text', text, x, y, color }`
- `{ type: 'symbol', symbol, x, y, color }`

All x/y normalized 0–1. No localStorage; in-memory like the rest of the app.

## Toolbar UI

Second row inside `.marker-tools` in the annotation modal:

- Pen button, Text button, text input, color picker (`#ff0000` default),
  width slider (1–20, default 4).
- Five symbol stamp buttons, each with a color swatch matching its symbol.
- Tool state: `activeTool = 'marker' | 'pen' | 'text' | 'symbol'`;
  `'marker'` is default so current behavior is unchanged. Selecting a symbol
  sets `activeSymbol`.
- Styles reuse `:root` tokens; responsive rules at index.html:304–327 extended.

## Behavior

- Pointer handlers branch on `activeTool`:
  - marker → existing `addMarkerFromCanvasEvent` (unchanged)
  - pen → pointerdown starts stroke, pointermove appends normalized points,
    pointerup/pointerleave commits
  - text → click places `#annText` input value (alert if empty)
  - symbol → click stamps `activeSymbol`
- Symbol stamps render survey-style: vector glyph + contrast halo + caption,
  no callout bubble. Symbol colors are forced from the symbol table (the color
  picker applies only to pen and text) so engineering color conventions hold.
- Undo: single shared history — each committed action (marker or drawing) is
  appended to a per-photo unified order; undo removes the most recent action
  of either kind. Clear-all clears both arrays.

## Rendering and export

- `drawAnnotationCanvas()` gains one pass after the markers loop:
  `photo.drawings.forEach(d => drawStroke(ctx, d, w, h))`.
- ZIP and Word exports reuse `drawAnnotationCanvas` via
  `getAnnotatedImageDataUrl`, so drawings are baked into exported JPEGs with
  no export-code changes.
- "Annotated" state (`updateButtons`, annotated count, card ribbon) treats a
  photo as annotated when `markers.length || drawings.length`.
- Word export equipment list: symbol stamps are listed alongside existing
  markers; pen strokes and text labels are not listed (they are visible in the
  image itself).

## Out of scope (YAGNI, matches survey behavior)

Redo, move/resize/delete of individual placed items, arrows, boxes,
measurement tools, persistence to disk.

## Validation

Manual browser verification on branch 2x: load photos, place each tool type,
verify undo across mixed marker/drawing history, verify ZIP and Word exports
contain baked drawings, verify marker-only flow unchanged.
