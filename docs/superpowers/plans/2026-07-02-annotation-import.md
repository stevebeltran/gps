# Annotation Import (survey → gps) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add freehand pen, text labels, and five engineering symbol stamps (ported from `Monster/survey/index.html`) to the gps app's photo annotation modal, alongside the existing bubble-marker system.

**Architecture:** Single-file app (`gps/index.html`). Each photo gains a `drawings:[]` array parallel to `markers:[]`, plus an `annHistory:[]` unified undo order. All new tools write normalized-0–1 stroke objects. One extra draw pass in `drawAnnotationCanvas()` renders drawings on-screen and in ZIP/Word exports (both reuse that function).

**Tech Stack:** Vanilla JS, Canvas 2D. No new dependencies.

## Global Constraints

- All edits confined to `gps/index.html` (plus this docs folder). No new files, no new CDN deps.
- Coordinates on stored annotation objects are normalized 0–1 (fractions of canvas width/height).
- Existing bubble-marker behavior must not change; `'marker'` stays the default tool when the modal opens.
- Symbol stamp colors are forced from `DRAW_SYMBOL_COLORS` (engineering conventions); the color picker applies only to pen and text.
- No automated test infrastructure exists in this repo (static single-file HTML). Each task ends with a manual browser verification step instead of an automated test run.
- Spec: `docs/superpowers/specs/2026-07-02-annotation-import-design.md`.

---

### Task 1: Data model, symbol tables, drawing primitives

**Files:**
- Modify: `index.html:762-763` (state), `index.html:885` and `index.html:890` (photo factories), after `drawMarker`'s helpers (~line 2669, near `roundedRect`)

**Interfaces:**
- Produces: globals `DRAW_SYMBOL_LABELS`, `DRAW_SYMBOL_COLORS`, `DRAW_SYMBOL_NAMES`, `activeTool`, `activeSymbol`, `currentStroke`, `penDrawing`; functions `contrastColor(hex)`, `drawSymbolIcon(c,type,cx,cy,size,color)`, `drawStroke(c,stroke,cw,ch)`; photo fields `drawings:[]`, `annHistory:[]`.

- [ ] **Step 1: Add state + tables** — after line 763 (`let activeMarkerType = ...`):

```js
  const DRAW_SYMBOL_LABELS = { power:"PWR", ethernet:"ETH", rf:"RF", crane:"CRANE", unistrut:"STRUT" };
  // Engineering color conventions: power=red (APWA electrical), ethernet=blue (TIA-606),
  // rf=orange (APWA comms), crane=safety yellow (OSHA), unistrut=galvanized gray.
  const DRAW_SYMBOL_COLORS = { power:"#e60000", ethernet:"#0057b8", rf:"#ff7a00", crane:"#ffd100", unistrut:"#9ea3a8" };
  const DRAW_SYMBOL_NAMES = { power:"Power", ethernet:"Ethernet", rf:"RF", crane:"Crane", unistrut:"Unistrut" };
  let activeTool = "marker", activeSymbol = "power", currentStroke = null, penDrawing = false;
```

- [ ] **Step 2: Extend photo factories** — in both return objects (lines 885 and 890) append `, drawings:[], annHistory:[]` after `markers:[]`.

- [ ] **Step 3: Port drawing primitives** — insert after `roundedRect` (line 2669), verbatim from survey (lines 21474–21579, 21612–21636) with two adaptations: table names prefixed `DRAW_`, text font uses `MARKUP_FONT`:

```js
  function contrastColor(hex){
    const c=hex.replace("#",""), r=parseInt(c.substr(0,2),16), g=parseInt(c.substr(2,2),16), b=parseInt(c.substr(4,2),16);
    return ((0.299*r+0.587*g+0.114*b)/255) > 0.6 ? "#000000" : "#ffffff";
  }
  function drawSymbolIcon(c,type,cx,cy,size,color){
    const s=size, base=Math.max(1.5,size*0.11), halo=Math.max(2,size*0.14), contrast=contrastColor(color);
    c.save(); c.lineCap="round"; c.lineJoin="round";
    const buildStroke=()=>{
      if(type==="ethernet"){
        c.beginPath(); c.rect(cx-s*0.7,cy-s*0.5,s*1.4,s); c.stroke();
        c.beginPath(); c.rect(cx-s*0.2,cy+s*0.5,s*0.4,s*0.3); c.stroke();
        for(let i=-2;i<=2;i++){ c.beginPath(); c.moveTo(cx+i*s*0.22,cy-s*0.5); c.lineTo(cx+i*s*0.22,cy-s*0.1); c.stroke(); }
      }else if(type==="rf"){
        c.beginPath(); c.moveTo(cx,cy+s); c.lineTo(cx,cy-s*0.2); c.stroke();
        c.beginPath(); c.moveTo(cx-s*0.4,cy-s); c.lineTo(cx,cy-s*0.2); c.lineTo(cx+s*0.4,cy-s); c.stroke();
        for(let r=1;r<=3;r++){ c.beginPath(); c.arc(cx,cy-s*0.2,s*0.3*r,-Math.PI*0.85,-Math.PI*0.15); c.stroke(); }
      }else if(type==="crane"){
        c.beginPath(); c.moveTo(cx-s*0.5,cy+s); c.lineTo(cx-s*0.5,cy-s*0.6); c.stroke();
        c.beginPath(); c.moveTo(cx-s,cy-s*0.6); c.lineTo(cx+s,cy-s*0.6); c.stroke();
        c.beginPath(); c.moveTo(cx+s*0.6,cy-s*0.6); c.lineTo(cx+s*0.6,cy+s*0.2); c.stroke();
        c.beginPath(); c.moveTo(cx-s,cy-s*0.6); c.lineTo(cx-s*0.5,cy-s*0.6-s*0.4); c.lineTo(cx-s*0.5,cy-s*0.6); c.stroke();
      }else if(type==="unistrut"){
        c.beginPath(); c.moveTo(cx+s*0.6,cy-s*0.7); c.lineTo(cx-s*0.6,cy-s*0.7); c.lineTo(cx-s*0.6,cy+s*0.7); c.lineTo(cx+s*0.6,cy+s*0.7); c.stroke();
        c.beginPath(); c.moveTo(cx+s*0.6,cy-s*0.7); c.lineTo(cx+s*0.6,cy-s*0.35); c.moveTo(cx+s*0.6,cy+s*0.7); c.lineTo(cx+s*0.6,cy+s*0.35); c.stroke();
      }
    };
    if(type==="power"){
      c.beginPath();
      c.moveTo(cx+s*0.15,cy-s); c.lineTo(cx-s*0.35,cy+s*0.1); c.lineTo(cx+s*0.05,cy+s*0.1);
      c.lineTo(cx-s*0.15,cy+s); c.lineTo(cx+s*0.4,cy-s*0.2); c.lineTo(cx,cy-s*0.2); c.closePath();
      c.lineWidth=halo*1.6; c.strokeStyle=contrast; c.stroke(); c.fillStyle=color; c.fill();
    }else{
      c.strokeStyle=contrast; c.lineWidth=base+halo; buildStroke();
      c.strokeStyle=color; c.lineWidth=base; buildStroke();
    }
    const fontPx=Math.max(9,size*0.6);
    c.font=`bold ${fontPx}px ${MARKUP_FONT}`; c.textAlign="center"; c.textBaseline="top";
    const label=DRAW_SYMBOL_LABELS[type] || type;
    c.lineWidth=Math.max(2,fontPx*0.3); c.strokeStyle=contrast; c.strokeText(label,cx,cy+s*1.15);
    c.fillStyle=color; c.fillText(label,cx,cy+s*1.15);
    c.restore();
  }
  function drawStroke(c,stroke,cw,ch){
    if(stroke.type==="pen"){
      if(!stroke.points || !stroke.points.length) return;
      c.save(); c.strokeStyle=stroke.color; c.lineWidth=stroke.width; c.lineCap="round"; c.lineJoin="round";
      c.beginPath();
      stroke.points.forEach((p,i)=>{ const x=p.x*cw, y=p.y*ch; if(i===0) c.moveTo(x,y); else c.lineTo(x,y); });
      c.stroke(); c.restore();
    }else if(stroke.type==="text"){
      c.save(); c.fillStyle=stroke.color; c.textAlign="left"; c.textBaseline="alphabetic";
      c.font=`700 ${Math.max(12,ch*0.03)}px ${MARKUP_FONT}`;
      c.fillText(stroke.text,stroke.x*cw,stroke.y*ch); c.restore();
    }else if(stroke.type==="symbol"){
      drawSymbolIcon(c,stroke.symbol,stroke.x*cw,stroke.y*ch,ch*0.045,stroke.color);
    }
  }
```

- [ ] **Step 4: Verify** — open index.html in browser, confirm no console errors on load.
- [ ] **Step 5: Commit** — `git add index.html && git commit -m "feat: add drawing primitives and data model for imported annotation tools"`

---

### Task 2: Toolbar UI (HTML + CSS)

**Files:**
- Modify: `index.html:684-685` (markup between `.modal-head` and `.canvas-wrap`), `index.html:302` area (CSS), `index.html:687` (hint text)

**Interfaces:**
- Produces: DOM ids `drawTools`, `drawTextInput`, `drawColor`, `drawWidth`; buttons with `data-tool` (`pen`|`text`|`symbol`) and `data-symbol` attributes, class `.draw-btn`. Task 3 wires these.

- [ ] **Step 1: Insert toolbar row** — between `</div>` closing `.modal-head` (line 684) and `.canvas-wrap` (line 685):

```html
      <div id="drawTools" class="draw-tools">
        <span class="draw-group-label">Draw</span>
        <button class="draw-btn" type="button" data-tool="pen">✏️ Pen</button>
        <button class="draw-btn" type="button" data-tool="text">Text</button>
        <input type="text" id="drawTextInput" placeholder="Text to place…">
        <label class="draw-label">Color <input type="color" id="drawColor" value="#ff0000"></label>
        <label class="draw-label">Width <input type="range" id="drawWidth" min="1" max="20" value="4"></label>
        <span class="draw-sep"></span>
        <span class="draw-group-label">Stamps</span>
        <button class="draw-btn draw-symbol" type="button" data-tool="symbol" data-symbol="power" style="border-left:6px solid #e60000">⚡ Power</button>
        <button class="draw-btn draw-symbol" type="button" data-tool="symbol" data-symbol="ethernet" style="border-left:6px solid #0057b8">🔌 Ethernet</button>
        <button class="draw-btn draw-symbol" type="button" data-tool="symbol" data-symbol="rf" style="border-left:6px solid #ff7a00">📶 RF</button>
        <button class="draw-btn draw-symbol" type="button" data-tool="symbol" data-symbol="crane" style="border-left:6px solid #ffd100">🏗 Crane</button>
        <button class="draw-btn draw-symbol" type="button" data-tool="symbol" data-symbol="unistrut" style="border-left:6px solid #9ea3a8">▯ Unistrut</button>
      </div>
```

- [ ] **Step 2: Add CSS** — after `.hint` rule (line 302):

```css
    .draw-tools{display:flex;flex-wrap:wrap;align-items:center;gap:8px;padding:8px 16px;border-bottom:1px solid var(--stroke);background:#fff}
    .draw-group-label{font-size:.68rem;font-weight:800;color:var(--muted);text-transform:uppercase;letter-spacing:.04em}
    .draw-btn{display:inline-flex;align-items:center;gap:6px;min-height:38px;padding:6px 12px;background:#fff;color:#1f2937;border:1px solid var(--stroke);border-radius:10px;font-size:.78rem;font-weight:700;cursor:pointer}
    .draw-btn.active{outline:3px solid rgba(37,99,235,.12);border-color:var(--accent);background:#eff6ff}
    .draw-tools input[type="text"]{min-height:38px;padding:6px 10px;border:1px solid var(--stroke);border-radius:10px;font-size:.78rem;min-width:150px}
    .draw-label{display:inline-flex;align-items:center;gap:6px;font-size:.74rem;font-weight:700;color:var(--muted)}
    .draw-sep{width:1px;align-self:stretch;background:var(--stroke)}
```

- [ ] **Step 3: Update hint** (line 687) to: `Select a marker type and click the photo to place a callout bubble — or use the Draw row for freehand pen, text labels, and symbol stamps. Undo removes the most recent addition of either kind. Use Save when finished.`

- [ ] **Step 4: Verify** — open modal in browser; new row renders below header, buttons styled like the rest of the app, responsive wrap OK at narrow width.
- [ ] **Step 5: Commit** — `git commit -am "feat: add draw-tools toolbar row to annotation modal"`

---

### Task 3: Tool selection, pointer handlers, unified undo

**Files:**
- Modify: `index.html:785-815` (event wiring + `addMarkerFromCanvasEvent`), `index.html:2426` (`renderMarkerPalette`), `index.html:2427-2440` (`openAnnotationModal`)

**Interfaces:**
- Consumes: Task 1 globals/functions, Task 2 DOM.
- Produces: `setActiveDrawTool(tool, symbol)`, `handleCanvasPointerDown(e)`, `handleCanvasPointerMove(e)`, `endPenStroke()`, `commitDrawing(stroke)`, `refreshAfterAnnotationChange()`. `annHistory` entries `"marker"`/`"drawing"` pushed on every commit.

- [ ] **Step 1: Replace canvas wiring** (lines 790–791) with:

```js
  canvas.addEventListener("pointerdown", handleCanvasPointerDown);
  canvas.addEventListener("pointermove", handleCanvasPointerMove);
  canvas.addEventListener("pointerup", endPenStroke);
  canvas.addEventListener("pointerleave", endPenStroke);
  canvas.addEventListener("click", e => { if(!window.PointerEvent && activeTool === "marker") addMarkerFromCanvasEvent(e); });
  document.querySelectorAll("#drawTools .draw-btn").forEach(btn => btn.addEventListener("click", () => setActiveDrawTool(btn.dataset.tool, btn.dataset.symbol)));
```

- [ ] **Step 2: Replace undo handler** (line 787) with:

```js
  undoMarkerBtn.addEventListener("click", () => {
    if(!activePhoto || !activePhoto.annHistory) return;
    const last = activePhoto.annHistory.pop();
    if(last === "drawing") activePhoto.drawings.pop();
    else if(last === "marker") activePhoto.markers.pop();
    else return;
    drawAnnotationCanvas(); render();
  });
```

- [ ] **Step 3: Add tool/pointer functions** — after `addMarkerFromCanvasEvent` (line 815):

```js
  function setActiveDrawTool(tool, symbol){
    activeTool = tool;
    if(tool === "symbol" && symbol) activeSymbol = symbol;
    document.querySelectorAll("#drawTools .draw-btn").forEach(b => {
      const isActive = b.dataset.tool === activeTool && (activeTool !== "symbol" || b.dataset.symbol === activeSymbol);
      b.classList.toggle("active", isActive);
    });
    renderMarkerPalette();
    if(tool === "text"){ const t = $("drawTextInput"); if(t) t.focus(); }
  }
  function getNormalizedCanvasPos(e){
    const rect = canvas.getBoundingClientRect();
    if(!rect.width || !rect.height) return null;
    const clientX = Number.isFinite(e.clientX) ? e.clientX : (e.touches && e.touches[0] ? e.touches[0].clientX : rect.left + rect.width / 2);
    const clientY = Number.isFinite(e.clientY) ? e.clientY : (e.touches && e.touches[0] ? e.touches[0].clientY : rect.top + rect.height / 2);
    return { x:(clientX - rect.left) / rect.width, y:(clientY - rect.top) / rect.height };
  }
  function refreshAfterAnnotationChange(){
    try{ drawAnnotationCanvas(); renderMetrics(); renderPreviews(); renderGroups(); updateButtons(); }
    catch(err){ console.error("Annotation draw failed", err); setStatus(`Annotation draw failed: ${err && err.message ? err.message : "unknown error"}`, false); }
  }
  function commitDrawing(stroke){
    if(!activePhoto) return;
    if(!activePhoto.drawings) activePhoto.drawings = [];
    if(!activePhoto.annHistory) activePhoto.annHistory = [];
    activePhoto.drawings.push(stroke);
    activePhoto.annHistory.push("drawing");
    refreshAfterAnnotationChange();
  }
  function handleCanvasPointerDown(e){
    if(!activePhoto || !activeImage) return;
    if(activeTool === "marker"){ addMarkerFromCanvasEvent(e); return; }
    if(e && e.preventDefault) e.preventDefault();
    const pos = getNormalizedCanvasPos(e);
    if(!pos) return;
    if(activeTool === "text"){
      const text = ($("drawTextInput").value || "").trim();
      if(!text){ setStatus('Type the label in the "Text to place" box first, then click the photo.', false); return; }
      commitDrawing({ type:"text", text, x:pos.x, y:pos.y, color:$("drawColor").value });
      return;
    }
    if(activeTool === "symbol"){
      commitDrawing({ type:"symbol", symbol:activeSymbol, x:pos.x, y:pos.y, color:DRAW_SYMBOL_COLORS[activeSymbol] || $("drawColor").value });
      return;
    }
    if(activeTool === "pen"){
      penDrawing = true;
      currentStroke = { type:"pen", color:$("drawColor").value, width:parseInt($("drawWidth").value, 10) || 4, points:[pos] };
    }
  }
  function handleCanvasPointerMove(e){
    if(!penDrawing || activeTool !== "pen" || !currentStroke) return;
    const pos = getNormalizedCanvasPos(e);
    if(!pos) return;
    currentStroke.points.push(pos);
    drawAnnotationCanvas();
  }
  function endPenStroke(){
    if(penDrawing && currentStroke && currentStroke.points.length > 1){
      const stroke = currentStroke;
      currentStroke = null; penDrawing = false;
      commitDrawing(stroke);
      return;
    }
    currentStroke = null; penDrawing = false;
  }
```

- [ ] **Step 4: Marker history + palette interplay:**
  - In `addMarkerFromCanvasEvent` (line 804), after `activePhoto.markers.push(...)` add: `if(!activePhoto.annHistory) activePhoto.annHistory = []; activePhoto.annHistory.push("marker");`
  - In `renderMarkerPalette` (line 2426): active class condition becomes `o.id===activeMarkerType && activeTool==="marker"`; palette button click handler body becomes `activeMarkerType=b.dataset.markerId; setActiveDrawTool("marker");`.
  - In `openAnnotationModal` (after line 2431) add `setActiveDrawTool("marker");` and `$("drawTextInput").value = "";` so each photo opens in marker mode.

- [ ] **Step 5: Verify** — browser: place markers (unchanged), draw pen strokes with color/width, place text, stamp each symbol, undo across mixed history in reverse order.
- [ ] **Step 6: Commit** — `git commit -am "feat: wire pen/text/symbol tools with unified undo"`

---

### Task 4: Rendering, exports, annotated-state checks

**Files:**
- Modify: `index.html:2460-2467` (`drawAnnotationCanvas`), `index.html:1945`, `2160-2161`, `2190`, `2223`, `2682`, `2703`, `2723`, `2739`, `2753`, `2770`, `2796`, `2907` (annotated checks), `index.html:2785` (Word marker list)

**Interfaces:**
- Consumes: `drawStroke`, `currentStroke`, `DRAW_SYMBOL_LABELS`, `DRAW_SYMBOL_NAMES`.
- Produces: `hasAnnotations(photo)` used by all annotated-state checks.

- [ ] **Step 1: Extend `drawAnnotationCanvas`** — after the `photo.markers.forEach(...)` line add:

```js
    (photo.drawings || []).forEach(d => drawStroke(c, d, targetCanvas.width, targetCanvas.height));
    if(targetCanvas === canvas && currentStroke) drawStroke(c, currentStroke, targetCanvas.width, targetCanvas.height);
```

- [ ] **Step 2: Add helper** — near `getMarkerOption` (line 2635):

```js
  function hasAnnotations(p){ return !!((p.markers && p.markers.length) || (p.drawings && p.drawings.length)); }
```

- [ ] **Step 3: Replace annotated-state checks** — swap `p.markers.length` → `hasAnnotations(p)` in exactly these annotated-or-not contexts (leave marker-specific logic alone):
  - 1945 `renderMetrics`: `photos.filter(p=>hasAnnotations(p)).length`
  - 2160 `annotatedClass`: `hasAnnotations(p) ? " annotated" : ""`
  - 2161 count text — replace with:
    ```js
    const markerCount = p.markers.length, drawingCount = (p.drawings || []).length;
    const countParts = [];
    if(markerCount) countParts.push(`${markerCount} marker${markerCount===1?"":"s"}`);
    if(drawingCount) countParts.push(`${drawingCount} drawing${drawingCount===1?"":"s"}`);
    const markerCountText = countParts.join(" · ");
    ```
  - 2190 `docGroups`: `g.photos.some(p=>hasAnnotations(p))`
  - 2223 `getDirectorySubfoldersForGroup`: `if(hasAnnotations(p)){`
  - 2682, 2703 (`downloadZip`), 2723 (`downloadWordDocument`), 2739, 2753, 2770, 2796 (`buildWordDocumentBlob`): same substitution
  - 2907 `updateButtons`: `docDisabled=photos.filter(p=>hasAnnotations(p)).length===0`

- [ ] **Step 4: Word export lists** — replace line 2785 with:

```js
        if(p.markers.length) children.push(metaLine("Markers",p.markers.map((m,i)=>`${i+1}. ${getMarkerOption(m.type).symbol} ${getMarkerOption(m.type).label}`).join("; ")));
        const symbolStamps=(p.drawings||[]).filter(d=>d.type==="symbol");
        if(symbolStamps.length) children.push(metaLine("Symbols",symbolStamps.map((d,i)=>`${i+1}. ${DRAW_SYMBOL_LABELS[d.symbol] || d.symbol} ${DRAW_SYMBOL_NAMES[d.symbol] || ""}`.trim()).join("; ")));
```

- [ ] **Step 5: Verify** — browser: drawings appear in modal; card shows "N drawings" + orange ANNOTATED state with drawings only; ZIP 05_Annotated contains baked drawings; Word doc embeds drawings + Symbols line; Word button enables with drawings only.
- [ ] **Step 6: Commit** — `git commit -am "feat: render drawings in canvas and exports, count in annotated state"`

---

### Task 5: End-to-end manual verification

- [ ] Load a folder of photos (mixed GPS / no-GPS).
- [ ] Marker-only flow identical to before (bubbles, undo, exports).
- [ ] Pen: multiple strokes, different colors/widths; live preview while dragging; resize window mid-session — strokes stay anchored (normalized coords).
- [ ] Text: empty-input guard message; placed text scales with canvas.
- [ ] Symbols: all five render with halo + caption; forced colors ignore picker.
- [ ] Undo: alternate marker/pen/symbol placements, undo unwinds in exact reverse order.
- [ ] ZIP export: `05_Annotated` JPEGs include drawings at full resolution.
- [ ] Word export: images baked, "Markers" and "Symbols" lines correct; a photo with only drawings is included.
- [ ] Commit any fixes; final commit.
