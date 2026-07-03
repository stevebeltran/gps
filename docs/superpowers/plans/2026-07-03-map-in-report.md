# Map in Word Report Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Embed a satellite overview map and per-location street maps of plotted photo GPS positions into the exported Word report, with a deterministic all-or-nothing tile renderer and an offline schematic fallback.

**Architecture:** Single-file browser app (`index.html`, one IIFE). Two new pure-ish async functions produce PNG data URLs (`buildMapImageDataUrl` from OSM/Esri tiles, `buildSchematicMapDataUrl` offline), a wrapper picks between them, and `buildWordDocumentBlob` calls the wrapper at two insertion points. No new libraries.

**Tech Stack:** Vanilla JS, Canvas 2D, Web Mercator tile math, docx 8.5 UMD (already loaded), OSM + Esri World Imagery public tile endpoints.

**Spec:** `docs/superpowers/specs/2026-07-03-map-in-report-design.md`

## Global Constraints

- Branch: `map-in-report`. All code edits in `index.html` only. Commit only `index.html` per code task (`git add index.html`) — working tree has unrelated untracked files.
- Tile fetch is **all-or-nothing**: compose only after every tile resolves; any tile failing 2 attempts (8 s timeout each) → throw. No partial maps.
- Tile caps: zoom ≤ 19, ≥ 3, tile count ≤ 30.
- Styles fixed: overview = `"satellite"` (Esri), per-location = `"street"` (OSM). No new UI controls.
- Attribution text verbatim: street `© OpenStreetMap contributors`, satellite `Esri, Maxar, Earthstar Geographics`.
- Fallback canvas note verbatim: `Basemap unavailable — schematic plot`.
- Legend line verbatim: `Marker numbers correspond to figure numbers.`
- A map that fails even the fallback must NOT abort the report — skip that map.
- No test framework. "Verify" steps = JS parse check (`node -e "const s=require('fs').readFileSync('index.html','utf8');const m=s.match(/<script>([\s\S]*)<\/script>/);new Function(m[1]);console.log('JS parses OK')"`) plus the browser checks described in Task 4. Tasks 1–3 implementers run the parse check only; full browser verification is Task 4 (controller).

---

### Task 1: Tile map renderer

**Files:**
- Modify: `index.html` — add new functions inside the IIFE, directly BEFORE the line `async function getAnnotatedImageDataUrl(photo,maxWidth=960,quality=.72){` (currently index.html:2883)

**Interfaces:**
- Consumes: `loadImage(src)` (existing, index.html:2884), `clamp(v,min,max)` (existing).
- Produces: `buildMapImageDataUrl(points, style)` → `Promise<string>` (PNG data URL); throws `Error` on tile failure. `points`: `[{lat:Number, lon:Number, label:String}]`, `style`: `"street"|"satellite"`. Also shared helpers `drawMapMarker(ctx,x,y,label)`, `drawMapScaleBar(ctx,canvasH,metersPerPixel)`, `drawMapAttribution(ctx,canvasW,canvasH,text)` — Task 2 reuses all three.

- [ ] **Step 1: Add the map renderer code**

Insert this block before `async function getAnnotatedImageDataUrl` (keep it as one contiguous block):

```js
  const MAP_TARGET_W=1000, MAP_TARGET_H=700, MAP_MAX_TILES=30, MAP_TILE_TIMEOUT_MS=8000;
  const MAP_STYLES={
    street:{ url:(z,x,y,i)=>`https://${"abc"[i%3]}.tile.openstreetmap.org/${z}/${x}/${y}.png`, attribution:"© OpenStreetMap contributors" },
    satellite:{ url:(z,x,y)=>`https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/${z}/${y}/${x}`, attribution:"Esri, Maxar, Earthstar Geographics" }
  };
  function lonToTileX(lon,z){ return (lon+180)/360*Math.pow(2,z); }
  function latToTileY(lat,z){ const r=lat*Math.PI/180; return (1-Math.log(Math.tan(r)+1/Math.cos(r))/Math.PI)/2*Math.pow(2,z); }
  function mapMetersPerPixel(lat,z){ return 156543.03392*Math.cos(lat*Math.PI/180)/Math.pow(2,z); }
  function loadTileImage(url){
    return new Promise((resolve,reject)=>{
      const img=new Image(); img.crossOrigin="anonymous";
      const timer=setTimeout(()=>{ img.src=""; reject(new Error(`Tile timed out: ${url}`)); },MAP_TILE_TIMEOUT_MS);
      img.onload=()=>{ clearTimeout(timer); resolve(img); };
      img.onerror=()=>{ clearTimeout(timer); reject(new Error(`Tile failed: ${url}`)); };
      img.src=url;
    });
  }
  async function loadTileWithRetry(url){
    try{ return await loadTileImage(url); }
    catch{ return loadTileImage(url); }
  }
  function mapBoundsForPoints(points){
    let minLat=Infinity,maxLat=-Infinity,minLon=Infinity,maxLon=-Infinity;
    for(const p of points){ minLat=Math.min(minLat,p.lat); maxLat=Math.max(maxLat,p.lat); minLon=Math.min(minLon,p.lon); maxLon=Math.max(maxLon,p.lon); }
    const midLat=(minLat+maxLat)/2;
    const minLatSpan=120/111320, minLonSpan=120/(111320*Math.max(.2,Math.cos(midLat*Math.PI/180)));
    if(maxLat-minLat<minLatSpan){ const c=(minLat+maxLat)/2; minLat=c-minLatSpan/2; maxLat=c+minLatSpan/2; }
    if(maxLon-minLon<minLonSpan){ const c=(minLon+maxLon)/2; minLon=c-minLonSpan/2; maxLon=c+minLonSpan/2; }
    const latPad=(maxLat-minLat)*.15, lonPad=(maxLon-minLon)*.15;
    return { minLat:minLat-latPad, maxLat:maxLat+latPad, minLon:minLon-lonPad, maxLon:maxLon+lonPad };
  }
  function pickMapZoom(bounds){
    for(let z=19;z>=3;z--){
      const pxW=(lonToTileX(bounds.maxLon,z)-lonToTileX(bounds.minLon,z))*256;
      const pxH=(latToTileY(bounds.minLat,z)-latToTileY(bounds.maxLat,z))*256;
      if(pxW>MAP_TARGET_W || pxH>MAP_TARGET_H) continue;
      const cx=(lonToTileX(bounds.minLon,z)+lonToTileX(bounds.maxLon,z))/2*256;
      const cy=(latToTileY(bounds.maxLat,z)+latToTileY(bounds.minLat,z))/2*256;
      const tx0=Math.floor((cx-MAP_TARGET_W/2)/256), tx1=Math.floor((cx+MAP_TARGET_W/2)/256);
      const ty0=Math.floor((cy-MAP_TARGET_H/2)/256), ty1=Math.floor((cy+MAP_TARGET_H/2)/256);
      if((tx1-tx0+1)*(ty1-ty0+1)<=MAP_MAX_TILES) return z;
    }
    return 3;
  }
  function drawMapMarker(ctx,x,y,label){
    ctx.save();
    ctx.beginPath(); ctx.moveTo(x,y); ctx.lineTo(x,y-10); ctx.strokeStyle="#1d4ed8"; ctx.lineWidth=3; ctx.stroke();
    ctx.beginPath(); ctx.arc(x,y-22,13,0,Math.PI*2);
    ctx.fillStyle="#2563eb"; ctx.fill();
    ctx.lineWidth=2.5; ctx.strokeStyle="#ffffff"; ctx.stroke();
    ctx.fillStyle="#ffffff"; ctx.font="bold 14px Arial,sans-serif"; ctx.textAlign="center"; ctx.textBaseline="middle";
    ctx.fillText(String(label),x,y-21.5);
    ctx.beginPath(); ctx.arc(x,y,3,0,Math.PI*2); ctx.fillStyle="#1d4ed8"; ctx.fill();
    ctx.restore();
  }
  function drawMapScaleBar(ctx,canvasH,metersPerPixel){
    const nice=[5,10,20,50,100,200,500,1000,2000,5000,10000];
    const targetPx=170;
    let meters=nice[0];
    for(const n of nice){ if(n/metersPerPixel<=targetPx) meters=n; }
    const px=meters/metersPerPixel;
    const x=16, y=canvasH-20;
    ctx.save();
    ctx.fillStyle="rgba(255,255,255,.85)"; ctx.fillRect(x-6,y-24,px+12,32);
    ctx.strokeStyle="#111827"; ctx.lineWidth=2;
    ctx.beginPath(); ctx.moveTo(x,y-6); ctx.lineTo(x,y); ctx.lineTo(x+px,y); ctx.lineTo(x+px,y-6); ctx.stroke();
    const feet=Math.round(meters*3.28084);
    ctx.fillStyle="#111827"; ctx.font="11px Arial,sans-serif"; ctx.textAlign="center"; ctx.textBaseline="bottom";
    ctx.fillText(`${meters>=1000?meters/1000+" km":meters+" m"} / ${feet>=5280?(feet/5280).toFixed(1)+" mi":feet+" ft"}`,x+px/2,y-6);
    ctx.restore();
  }
  function drawMapAttribution(ctx,canvasW,canvasH,text){
    ctx.save();
    ctx.font="10px Arial,sans-serif"; ctx.textAlign="right"; ctx.textBaseline="bottom";
    const w=ctx.measureText(text).width;
    ctx.fillStyle="rgba(255,255,255,.8)"; ctx.fillRect(canvasW-w-12,canvasH-16,w+10,14);
    ctx.fillStyle="#374151"; ctx.fillText(text,canvasW-7,canvasH-4);
    ctx.restore();
  }
  async function buildMapImageDataUrl(points,style){
    if(!points || !points.length) throw new Error("No GPS points for map");
    const styleDef=MAP_STYLES[style] || MAP_STYLES.street;
    const bounds=mapBoundsForPoints(points);
    const z=pickMapZoom(bounds);
    const cx=(lonToTileX(bounds.minLon,z)+lonToTileX(bounds.maxLon,z))/2*256;
    const cy=(latToTileY(bounds.maxLat,z)+latToTileY(bounds.minLat,z))/2*256;
    const originX=cx-MAP_TARGET_W/2, originY=cy-MAP_TARGET_H/2;
    const tx0=Math.floor(originX/256), tx1=Math.floor((originX+MAP_TARGET_W)/256);
    const ty0=Math.floor(originY/256), ty1=Math.floor((originY+MAP_TARGET_H)/256);
    const maxTile=Math.pow(2,z)-1;
    const jobs=[];
    let ti=0;
    for(let tx=tx0;tx<=tx1;tx++) for(let ty=ty0;ty<=ty1;ty++){
      const cxT=clamp(tx,0,maxTile), cyT=clamp(ty,0,maxTile);
      jobs.push(loadTileWithRetry(styleDef.url(z,cxT,cyT,ti++)).then(img=>({img,tx,ty})));
    }
    const tiles=await Promise.all(jobs);
    const canvas=document.createElement("canvas"); canvas.width=MAP_TARGET_W; canvas.height=MAP_TARGET_H;
    const ctx=canvas.getContext("2d");
    ctx.fillStyle="#dbe4ee"; ctx.fillRect(0,0,MAP_TARGET_W,MAP_TARGET_H);
    for(const t of tiles) ctx.drawImage(t.img,Math.round(t.tx*256-originX),Math.round(t.ty*256-originY));
    for(const p of points) drawMapMarker(ctx,lonToTileX(p.lon,z)*256-originX,latToTileY(p.lat,z)*256-originY,p.label);
    const midLat=(bounds.minLat+bounds.maxLat)/2;
    drawMapScaleBar(ctx,MAP_TARGET_H,mapMetersPerPixel(midLat,z));
    drawMapAttribution(ctx,MAP_TARGET_W,MAP_TARGET_H,styleDef.attribution);
    return canvas.toDataURL("image/png");
  }
```

- [ ] **Step 2: Run the JS parse check**

Run the parse check from Global Constraints. Expected: `JS parses OK`.

- [ ] **Step 3: Self-review diff**

`git diff index.html` — confirm the block landed inside the IIFE (before `getAnnotatedImageDataUrl`), no duplicated helper names (grep: `lonToTileX` appears exactly once as a definition).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add deterministic tile map renderer"
```

---

### Task 2: Schematic fallback + wrapper + figure inserter

**Files:**
- Modify: `index.html` — add directly AFTER the Task 1 block (after `buildMapImageDataUrl`'s closing brace), still before `async function getAnnotatedImageDataUrl`

**Interfaces:**
- Consumes: `drawMapMarker`, `drawMapScaleBar`, `drawMapAttribution`, `MAP_TARGET_W`, `MAP_TARGET_H` (Task 1); `loadImage`, `dataUrlToUint8Array` (existing).
- Produces:
  - `buildSchematicMapDataUrl(points)` → `string` (PNG data URL, synchronous, no network, cannot reject).
  - `buildReportMapDataUrl(points, style)` → `Promise<{dataUrl:string, fallback:boolean}|null>` (null = both renderers failed; caller skips map).
  - `appendMapFigure(children, run, mapResult, caption, legend)` → `Promise<void>` — pushes ImageRun paragraph + caption (+ optional legend) onto the docx `children` array. `run` is the existing TextRun factory in `buildWordDocumentBlob` scope; docx classes are read from `window.docx` inside the function so it works outside that scope.

- [ ] **Step 1: Add the code**

```js
  function buildSchematicMapDataUrl(points){
    const W=MAP_TARGET_W,H=MAP_TARGET_H,PAD=90;
    const canvas=document.createElement("canvas"); canvas.width=W; canvas.height=H;
    const ctx=canvas.getContext("2d");
    ctx.fillStyle="#ffffff"; ctx.fillRect(0,0,W,H);
    ctx.strokeStyle="#e5e7eb"; ctx.lineWidth=1;
    for(let x=0;x<=W;x+=50){ ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,H); ctx.stroke(); }
    for(let y=0;y<=H;y+=50){ ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(W,y); ctx.stroke(); }
    const lat0=points.reduce((s,p)=>s+p.lat,0)/points.length, lon0=points.reduce((s,p)=>s+p.lon,0)/points.length;
    const cos0=Math.max(.2,Math.cos(lat0*Math.PI/180));
    const local=points.map(p=>({x:(p.lon-lon0)*111320*cos0, y:(p.lat-lat0)*110540, label:p.label}));
    let maxAbs=1;
    for(const p of local) maxAbs=Math.max(maxAbs,Math.abs(p.x),Math.abs(p.y));
    maxAbs=Math.max(maxAbs,60);
    const scale=Math.min((W/2-PAD)/maxAbs,(H/2-PAD)/maxAbs);
    for(const p of local) drawMapMarker(ctx,W/2+p.x*scale,H/2-p.y*scale,p.label);
    ctx.save();
    ctx.strokeStyle="#111827"; ctx.fillStyle="#111827"; ctx.lineWidth=2;
    const nx=W-46, ny=64;
    ctx.beginPath(); ctx.moveTo(nx,ny); ctx.lineTo(nx,ny-30); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(nx,ny-36); ctx.lineTo(nx-7,ny-22); ctx.lineTo(nx+7,ny-22); ctx.closePath(); ctx.fill();
    ctx.font="bold 14px Arial,sans-serif"; ctx.textAlign="center"; ctx.fillText("N",nx,ny+16);
    ctx.font="12px Arial,sans-serif"; ctx.textAlign="left"; ctx.fillStyle="#6b7280";
    ctx.fillText("Basemap unavailable — schematic plot",16,24);
    ctx.restore();
    drawMapScaleBar(ctx,H,1/scale);
    return canvas.toDataURL("image/png");
  }
  async function buildReportMapDataUrl(points,style){
    try{ return {dataUrl:await buildMapImageDataUrl(points,style),fallback:false}; }
    catch(err){
      console.warn("Tile map unavailable, using schematic",err);
      try{ return {dataUrl:buildSchematicMapDataUrl(points),fallback:true}; }
      catch(err2){ console.error("Schematic map failed",err2); return null; }
    }
  }
  async function appendMapFigure(children,run,mapResult,caption,legend){
    const api=window.docx||{}; const {Paragraph,ImageRun,AlignmentType}=api;
    if(!Paragraph||!ImageRun) return;
    const img=await loadImage(mapResult.dataUrl);
    const maxW=520, ratio=Math.min(1,maxW/img.naturalWidth);
    const width=Math.round(img.naturalWidth*ratio), height=Math.round(img.naturalHeight*ratio);
    children.push(new Paragraph({alignment:AlignmentType.CENTER,spacing:{before:80,after:40},children:[new ImageRun({data:dataUrlToUint8Array(mapResult.dataUrl),transformation:{width,height}})]}));
    children.push(new Paragraph({alignment:AlignmentType.CENTER,spacing:{after:legend?20:100},children:[run(caption,{italics:true,color:"595959",size:18})]}));
    if(legend) children.push(new Paragraph({alignment:AlignmentType.CENTER,spacing:{after:100},children:[run(legend,{color:"8a94a6",size:16})]}));
  }
```

- [ ] **Step 2: Run the JS parse check**

Expected: `JS parses OK`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add schematic map fallback and docx map figure helper"
```

---

### Task 3: Wire maps into the Word export

**Files:**
- Modify: `index.html` — `buildWordDocumentBlob` (TOC block ends at the `children.push(new Paragraph({children:[new PageBreak()]}));` on index.html:3012, section loop starts index.html:3014) and `downloadWordDocument` (~index.html:2935)

**Interfaces:**
- Consumes: `buildReportMapDataUrl`, `appendMapFigure` (Task 2); in-scope: `sorted`, `children`, `run`, `onProgress`, `idx`, `imgs`, `g`, `PageBreak`.
- Produces: `buildWordDocumentBlob` return gains `usedSchematic:boolean`.

- [ ] **Step 1: Overview map after the TOC**

Directly AFTER this existing line (index.html:3012):
```js
    children.push(new Paragraph({children:[new PageBreak()]}));
```
and BEFORE `let idx=0;`, insert:
```js
    let usedSchematic=false;
    const overviewPoints=sorted.map((g2,gi2)=>({lat:g2.centroidLat,lon:g2.centroidLon,label:String(gi2+1)})).filter(pt=>Number.isFinite(pt.lat)&&Number.isFinite(pt.lon));
    if(overviewPoints.length){
      onProgress("Building site maps...");
      const overviewMap=await buildReportMapDataUrl(overviewPoints,"satellite");
      if(overviewMap){
        if(overviewMap.fallback) usedSchematic=true;
        await appendMapFigure(children,run,overviewMap,"Site Overview Map",null);
        children.push(new Paragraph({children:[new PageBreak()]}));
      }
    }
```

- [ ] **Step 2: Per-location map at the top of each section**

Directly AFTER this existing line inside the `for(let gi=0;...)` loop (index.html:3020):
```js
      if(Number.isFinite(g.centroidLat) && Number.isFinite(g.centroidLon)) children.push(mapsLink(g.centroidLat,g.centroidLon));
```
and BEFORE `for(let pi=0;pi<imgs.length;pi++){`, insert:
```js
      const locPoints=imgs.map((p2,pi2)=>({lat:p2.lat,lon:p2.lon,label:String(idx+pi2+1)})).filter(pt=>Number.isFinite(pt.lat)&&Number.isFinite(pt.lon));
      if(locPoints.length){
        onProgress(`Building location map ${gi+1} of ${sorted.length}...`);
        const locMap=await buildReportMapDataUrl(locPoints,"street");
        if(locMap){
          if(locMap.fallback) usedSchematic=true;
          await appendMapFigure(children,run,locMap,`Location Map — ${g.name || "Unknown Location"}`,"Marker numbers correspond to figure numbers.");
        }
      }
```
(Label math: `idx` has not yet been incremented for this section's photos, and the photo loop below increments `idx` once per photo in `imgs` order — so photo `pi2` gets figure number `idx+pi2+1`. Filtering to GPS-bearing points happens after labeling, so numbers stay aligned with figures.)

- [ ] **Step 3: Return and surface the fallback flag**

Change the return of `buildWordDocumentBlob` (currently `return {blob,fileName:makeWordDocumentFileName(sorted)};`) to:
```js
    return {blob,fileName:makeWordDocumentFileName(sorted),usedSchematic};
```
In `downloadWordDocument`, change:
```js
      setStatus("Word document download ready.",false);
```
to:
```js
      setStatus(report.usedSchematic ? "Word document ready — basemap unavailable, schematic maps used." : "Word document download ready.",false);
```

- [ ] **Step 4: Run the JS parse check**

Expected: `JS parses OK`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: embed overview and per-location maps in Word export"
```

---

### Task 4: Verification (controller, browser)

**Files:**
- Create: scratchpad script `make-gps-jpeg.js` (NOT in the repo — write to the session scratchpad directory)
- Modify: none unless fixing regressions

- [ ] **Step 1: Generate a GPS-EXIF JPEG fixture**

Node script (scratchpad). It splices a hand-built EXIF APP1 segment (GPS IFD, 39.9471 N, 86.2743 W) into a hardcoded tiny baseline JPEG:

```js
const fs=require("fs");
// 1x1 white baseline JPEG
const baseJpeg=Buffer.from("/9j/4AAQSkZJRgABAQEAYABgAAD/2wBDAAgGBgcGBQgHBwcJCQgKDBQNDAsLDBkSEw8UHRofHh0aHBwgJC4nICIsIxwcKDcpLDAxNDQ0Hyc5PTgyPC4zNDL/wAALCAABAAEBAREA/8QAFAABAAAAAAAAAAAAAAAAAAAACf/EABQQAQAAAAAAAAAAAAAAAAAAAAD/2gAIAQEAAD8AVN//2Q==","base64");
function u16(v){ const b=Buffer.alloc(2); b.writeUInt16LE(v); return b; }
function u32(v){ const b=Buffer.alloc(4); b.writeUInt32LE(v); return b; }
function rational(num,den){ return Buffer.concat([u32(num),u32(den)]); }
function dmsRationals(deg){ const d=Math.floor(deg); const mFloat=(deg-d)*60; const m=Math.floor(mFloat); const s=Math.round((mFloat-m)*60*10000); return Buffer.concat([rational(d,1),rational(m,1),rational(s,10000)]); }
function entry(tag,type,count,valueOrOffset){ return Buffer.concat([u16(tag),u16(type),u32(count),valueOrOffset]); }
// TIFF little-endian. Layout: header(8) IFD0(2+1*12+4) GPSIFD(2+4*12+4) values
const tiffHeader=Buffer.concat([Buffer.from("II"),u16(42),u32(8)]);
const ifd0Offset=8, gpsIfdOffset=ifd0Offset+2+12+4;
const valuesOffset=gpsIfdOffset+2+4*12+4;
const latVal=dmsRationals(39.9471), lonVal=dmsRationals(86.2743);
const latOffset=valuesOffset, lonOffset=valuesOffset+24;
const ifd0=Buffer.concat([u16(1),entry(0x8825,4,1,u32(gpsIfdOffset)),u32(0)]);
const gpsIfd=Buffer.concat([
  u16(4),
  entry(0x0001,2,2,Buffer.from("N\0\0\0")),
  entry(0x0002,5,3,u32(latOffset)),
  entry(0x0003,2,2,Buffer.from("W\0\0\0")),
  entry(0x0004,5,3,u32(lonOffset)),
  u32(0)
]);
const tiff=Buffer.concat([tiffHeader,ifd0,gpsIfd,latVal,lonVal]);
const exifBody=Buffer.concat([Buffer.from("Exif\0\0"),tiff]);
const app1len=Buffer.alloc(2); app1len.writeUInt16BE(exifBody.length+2);
const app1=Buffer.concat([Buffer.from([0xFF,0xE1]),app1len,exifBody]);
// splice after SOI (first 2 bytes)
const out=Buffer.concat([baseJpeg.slice(0,2),app1,baseJpeg.slice(2)]);
fs.writeFileSync(process.argv[2]||"gps-fixture.jpg",out);
console.log("wrote",out.length,"bytes");
```

Run it, then verify the fixture parses: load in the preview page and confirm the app reports "With GPS: 1" after drop.

- [ ] **Step 2: Happy-path export**

Start the preview server (`.claude/launch.json`, config `gps-static`). Drop the GPS fixture (DataTransfer drop via preview_eval, as done for the professional-report verification). Annotate it (stub `getBoundingClientRect` if viewport is 0×0). Export Word. Expected: status reaches `Word document download ready.` (tiles fetched — machine has internet), no console errors, no `usedSchematic` message.

- [ ] **Step 3: Fallback-path export**

In-page before export: `window.__origImage=window.Image;` then break tile fetching by overriding the tile hosts — simplest reliable lever: `const realSrc=Object.getOwnPropertyDescriptor(HTMLImageElement.prototype,"src"); Object.defineProperty(HTMLImageElement.prototype,"src",{set(v){ if(/tile\.openstreetmap|arcgisonline/.test(v)){ realSrc.set.call(this,"http://127.0.0.1:9/x.png"); } else realSrc.set.call(this,v); }});`
Export Word again. Expected: status ends `Word document ready — basemap unavailable, schematic maps used.`, export completes, no uncaught errors. Reload page afterward to clear the override.

- [ ] **Step 4: No-GPS photos**

Drop a plain canvas JPEG (no EXIF). Annotate, export. Expected: no overview page, no location map, plain `Word document download ready.` — maps skipped silently.

- [ ] **Step 5: Human check (hand off to user)**

User exports a real survey and opens the .docx: overview map markers on correct buildings, per-location numbers match figures, scale bar + attribution visible.

- [ ] **Step 6: Update knowledge graph + finish**

Run `graphify update .`. Then finishing-a-development-branch skill.
