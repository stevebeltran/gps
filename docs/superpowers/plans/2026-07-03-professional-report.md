# Professional Survey Report Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Project Info panel, per-photo captions, and rebuild the Word export into an engineering-style document (cover page, linked TOC, headers/footers).

**Architecture:** Single-file browser app (`index.html`, ~3100 lines, one IIFE). All changes stay in that file. Project info persists in `localStorage`; captions live in-memory on photo objects like all other annotations. Word export uses the already-loaded `docx@8.5` UMD build (`window.docx`).

**Tech Stack:** Vanilla JS, docx 8.5 (CDN), JSZip, Leaflet. No build step, no test framework — every task ends with a manual browser verification step instead of an automated test run. Open `index.html` directly in Chrome to test.

**Spec:** `docs/superpowers/specs/2026-07-03-professional-report-design.md`

## Global Constraints

- Branch: `professional-report`. Do NOT touch `mobile.html`, `brinc_tank_gps_professional.html`, or the untracked zip packages.
- All edits in `index.html` only.
- `$` is the element helper: `const $ = id => document.getElementById(id);` (index.html:767).
- Empty project fields must render "—" in the Word doc, never blank.
- Status values exactly: `Draft`, `For Review`, `Issued`. Default revision exactly `"A"`.
- TOC must be a static bookmark-linked list (works on iPhone). NO `TableOfContents` field.
- Commit after every task. Commit only `index.html` (`git add index.html`) — the working tree has unrelated untracked files.

## Verification harness (used by every task)

No test framework exists. "Run test" steps mean: open `index.html` in Chrome (`start chrome "index.html"` from the repo dir), open DevTools console (F12), and check for errors plus the described behavior. For Word checks, drop 2+ GPS-tagged photos, annotate at least one (click photo → place a marker → Save), click "Download Word Document", open the .docx.

---

### Task 1: Project Info panel (UI + localStorage)

**Files:**
- Modify: `index.html` — HTML near line 610 (status toolbar), line 621 (agency input), line 767-768 (element consts), CSS `:root` block area (~line 374 `.agency-input`), JS bottom of IIFE (~line 3090)

**Interfaces:**
- Produces: `getProjectInfo()` → `{agency, pocName, pocTitle, pocPhone, pocEmail, siteAddress, jobNumber, installerName, installerTitle, installerPhone, installerEmail, revision, status, surveyDate}` (all strings, trimmed; `revision` defaults `"A"`, `status` defaults `"Draft"`). `effectiveSurveyDate(photoList)` → string. Tasks 3–5 consume both.
- Consumes: `$`, `formatSurveyDateDisplay`, `cleanFileNamePart` (all existing).

- [ ] **Step 1: Add panel HTML**

Insert directly AFTER the status toolbar `</div>` (line 612, after `<div class="toolbar status-toolbar">...</div>`), before `<section class="section-shell results-section">`:

```html
      <details id="projectInfoPanel" class="section-shell project-info" open>
        <summary class="pi-summary">Project Info — <span id="projectInfoBrief">not set</span></summary>
        <div class="pi-grid">
          <fieldset class="pi-block">
            <legend>Customer</legend>
            <label>Agency name <input id="piAgency" type="text" /></label>
            <label>POC name <input id="piPocName" type="text" /></label>
            <label>POC title <input id="piPocTitle" type="text" /></label>
            <label>POC phone <input id="piPocPhone" type="tel" /></label>
            <label>POC email <input id="piPocEmail" type="email" /></label>
            <label>Site address <input id="piSiteAddress" type="text" /></label>
            <label>Project / job # <input id="piJobNumber" type="text" /></label>
          </fieldset>
          <fieldset class="pi-block">
            <legend>BRINC (prepared by)</legend>
            <label>Installer name <input id="piInstallerName" type="text" /></label>
            <label>Title <input id="piInstallerTitle" type="text" /></label>
            <label>Phone <input id="piInstallerPhone" type="tel" /></label>
            <label>Email <input id="piInstallerEmail" type="email" /></label>
          </fieldset>
          <fieldset class="pi-block">
            <legend>Document</legend>
            <label>Revision <input id="piRevision" type="text" value="A" /></label>
            <label>Status
              <select id="piStatus">
                <option>Draft</option>
                <option>For Review</option>
                <option>Issued</option>
              </select>
            </label>
            <label>Survey date <input id="piSurveyDate" type="text" placeholder="Auto from photos" /></label>
            <button id="newProjectBtn" class="secondary-btn" type="button">New project</button>
          </fieldset>
        </div>
      </details>
```

- [ ] **Step 2: Remove old agency input**

Delete line 621:
```html
<input id="agencyNameInput" class="agency-input" type="text" placeholder="Agency name" aria-label="Agency name for export file names" />
```
Also delete the `.agency-input` CSS rule (~line 374) and remove `agencyNameInput` from the element-const declarations around line 768 (search `agencyNameInput=$("agencyNameInput")` or equivalent in the const list and remove that single declaration).

- [ ] **Step 3: Add panel CSS**

Add to the `<style>` block (near the other section styles):

```css
    .project-info{border:1px solid var(--stroke);border-radius:var(--radius);background:var(--panel);box-shadow:var(--shadow);padding:0}
    .pi-summary{cursor:pointer;padding:14px 18px;font-weight:800;font-size:.95rem;color:var(--text)}
    .pi-summary #projectInfoBrief{font-weight:600;color:var(--muted)}
    .pi-grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(240px,1fr));gap:14px;padding:0 18px 18px}
    .pi-block{border:1px solid var(--stroke3);border-radius:12px;padding:10px 14px 14px;display:grid;gap:8px;min-width:0}
    .pi-block legend{font-size:.78rem;font-weight:800;letter-spacing:.06em;text-transform:uppercase;color:var(--muted);padding:0 6px}
    .pi-block label{display:grid;gap:3px;font-size:.78rem;color:var(--muted)}
    .pi-block input,.pi-block select{min-height:34px;padding:4px 10px;border:1px solid var(--stroke);border-radius:8px;background:#fff;color:var(--text);font-size:.86rem}
    .pi-block #newProjectBtn{margin-top:4px;justify-self:start}
```

- [ ] **Step 4: Add project-info JS**

Add inside the IIFE, before `function setStatus` (~line 3082):

```js
  const PROJECT_INFO_KEY = "brincGpsProjectInfo";
  const PI_FIELDS = ["piAgency","piPocName","piPocTitle","piPocPhone","piPocEmail","piSiteAddress","piJobNumber","piInstallerName","piInstallerTitle","piInstallerPhone","piInstallerEmail","piRevision","piStatus","piSurveyDate"];
  const PI_CUSTOMER_FIELDS = ["piAgency","piPocName","piPocTitle","piPocPhone","piPocEmail","piSiteAddress","piJobNumber","piRevision","piStatus","piSurveyDate"];
  function getProjectInfo(){
    const v = id => { const el = $(id); return el ? el.value.trim() : ""; };
    return {
      agency:v("piAgency"), pocName:v("piPocName"), pocTitle:v("piPocTitle"), pocPhone:v("piPocPhone"), pocEmail:v("piPocEmail"),
      siteAddress:v("piSiteAddress"), jobNumber:v("piJobNumber"),
      installerName:v("piInstallerName"), installerTitle:v("piInstallerTitle"), installerPhone:v("piInstallerPhone"), installerEmail:v("piInstallerEmail"),
      revision:v("piRevision") || "A", status:v("piStatus") || "Draft", surveyDate:v("piSurveyDate")
    };
  }
  function effectiveSurveyDate(photoList){ const i = getProjectInfo(); return i.surveyDate || formatSurveyDateDisplay(photoList); }
  function saveProjectInfo(){
    try{
      const data = {};
      PI_FIELDS.forEach(id => { const el = $(id); if(el) data[id] = el.value; });
      localStorage.setItem(PROJECT_INFO_KEY, JSON.stringify(data));
    }catch{}
    updateProjectInfoBrief();
  }
  function loadProjectInfo(){
    try{
      const data = JSON.parse(localStorage.getItem(PROJECT_INFO_KEY) || "{}");
      PI_FIELDS.forEach(id => { const el = $(id); if(el && data[id] !== undefined) el.value = data[id]; });
    }catch{}
    updateProjectInfoBrief();
  }
  function updateProjectInfoBrief(){
    const i = getProjectInfo();
    const parts = [i.agency, i.jobNumber ? `Job #${i.jobNumber}` : "", `${i.status} Rev ${i.revision}`].filter(Boolean);
    const brief = $("projectInfoBrief");
    if(brief) brief.textContent = i.agency ? parts.join(" — ") : "not set";
  }
  function clearCustomerFields(){
    PI_CUSTOMER_FIELDS.forEach(id => { const el = $(id); if(!el) return; el.value = id === "piRevision" ? "A" : ""; });
    const st = $("piStatus"); if(st) st.value = "Draft";
    saveProjectInfo();
  }
  (function initProjectInfo(){
    const panel = $("projectInfoPanel");
    if(panel) panel.addEventListener("input", saveProjectInfo);
    const npb = $("newProjectBtn");
    if(npb) npb.addEventListener("click", clearCustomerFields);
    loadProjectInfo();
    if(panel && getProjectInfo().agency) panel.open = false;
  })();
```

- [ ] **Step 5: Repoint the two old `agencyNameInput` reads**

Line 2927 (`buildWordDocumentBlob`), replace:
```js
const agencyName=cleanFileNamePart(agencyNameInput ? agencyNameInput.value : "");
```
with:
```js
const agencyName=cleanFileNamePart(getProjectInfo().agency);
```

Line 3063 (`makeZipFileName`), replace:
```js
const agency = cleanFileNamePart(agencyNameInput ? agencyNameInput.value : "") || "Agency";
```
with:
```js
const agency = cleanFileNamePart(getProjectInfo().agency) || "Agency";
```

- [ ] **Step 6: Verify in browser**

Open `index.html` in Chrome. Expected: panel renders open with three fieldsets, no console errors. Type agency "Mesa PD" and job "2026-041" → summary line reads "Mesa PD — Job #2026-041 — Draft Rev A". Reload page → values persist AND panel loads collapsed (agency saved) showing only the summary line. Click "New project" → customer fields clear, Revision back to "A", Status "Draft", BRINC fields (fill one first) unchanged. Collapse/expand panel works.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add Project Info panel with localStorage persistence"
```

---

### Task 2: Photo captions

**Files:**
- Modify: `index.html` — `makePhoto` return (~line 993), `makeFallbackPhoto` return (~line 998), modal-foot HTML (~line 708), `openAnnotationModal` (~line 2539), save handler (line 814), `buildLocationSummaryText` (~line 2400)

**Interfaces:**
- Produces: `photo.caption` (string, default `""`) on every photo object. Task 5 consumes it for Word figure lines.
- Consumes: `$`, `activePhoto`, `closeAnnotationModal`, `render` (existing).

- [ ] **Step 1: Add `caption` to photo model**

In `makePhoto`'s return object (~line 993), change the tail `markers:[], drawings:[], annHistory:[] }` to `markers:[], drawings:[], annHistory:[], caption:"" }`.
Same change in `makeFallbackPhoto`'s return (~line 998).

- [ ] **Step 2: Add caption input to annotation modal**

In the modal foot (~line 708), insert BEFORE the `<div class="hint">` line:

```html
        <label class="caption-row">Caption <input id="captionInput" type="text" maxlength="200" placeholder="Shown under this photo in the Word report" /></label>
```

Add CSS near `.modal-foot` styles:

```css
    .caption-row{display:flex;align-items:center;gap:8px;font-size:.8rem;color:var(--muted);margin-bottom:6px}
    .caption-row input{flex:1;min-height:34px;padding:4px 10px;border:1px solid var(--stroke);border-radius:8px;font-size:.86rem}
```

- [ ] **Step 3: Load caption on modal open**

In `openAnnotationModal` (~line 2546), after `if(drawTextInput) drawTextInput.value = "";` add:

```js
    const captionInput = $("captionInput");
    if(captionInput) captionInput.value = activePhoto.caption || "";
```

- [ ] **Step 4: Save caption on Save click**

Replace line 814:
```js
  saveAnnotationBtn.addEventListener("click", () => { closeAnnotationModal(); render(); });
```
with:
```js
  saveAnnotationBtn.addEventListener("click", () => { if(activePhoto){ const ci=$("captionInput"); activePhoto.caption = ci ? ci.value.trim() : ""; } closeAnnotationModal(); render(); });
```
(`closeAnnotationModal` nulls `activePhoto`, so the caption write MUST come first.)

- [ ] **Step 5: Include caption in Location_Summary.txt**

In `buildLocationSummaryText` (~line 2400), replace the photo line:
```js
      lines.push(`- ${p.fileName} | ${formatDateForDisplay(p.capturedAt)} | ${gps} | ${tags} | ${p.matchedTimeReason || ""}`);
```
with:
```js
      lines.push(`- ${p.fileName} | ${formatDateForDisplay(p.capturedAt)} | ${gps} | ${tags} | ${p.matchedTimeReason || ""}${p.caption ? ` | Caption: ${p.caption}` : ""}`);
```

- [ ] **Step 6: Verify in browser**

Load photos, open one for annotation → caption box empty. Type "North wall conduit entry", Save. Reopen same photo → caption persists. Download ZIP → `Location_Summary.txt` photo line ends with `| Caption: North wall conduit entry`. No console errors.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add per-photo caption in annotation modal and ZIP summary"
```

---

### Task 3: Word export — cover page

**Files:**
- Modify: `index.html` — `buildWordDocumentBlob` (~lines 2912–2990)

**Interfaces:**
- Consumes: `getProjectInfo()`, `effectiveSurveyDate(photoList)` (Task 1), existing helpers `infoRow`, `infoCell`, `run`, `dataUrlToUint8Array`, `loadImage`.
- Produces: cover-page `children` prefix inside `buildWordDocumentBlob`; `info` (project info object) and `dash(v)` helper in that function's scope — Tasks 4 and 5 use both.

- [ ] **Step 1: Extend the docx destructure**

Line 2917, add `PageBreak` to the destructure:
```js
    const {Document,Packer,Paragraph,TextRun,ImageRun,AlignmentType,ExternalHyperlink,Table,TableRow,TableCell,WidthType,BorderStyle,Footer,PageNumber,PageBreak}=window.docx;
```

- [ ] **Step 2: Replace the old title block with a cover page**

Replace the current `children` initializer (lines 2929–2942, from `const children=[` through the closing `];` after the empty-paragraph spacer) with:

```js
    const info=getProjectInfo();
    const dash=v=>v||"—";
    const coverLogo=[];
    try{
      const logoEl=document.querySelector(".drop-brand img");
      if(logoEl && logoEl.src.startsWith("data:image")){
        const logoImg=await loadImage(logoEl.src);
        const lw=Math.min(220,logoImg.naturalWidth), lh=Math.round(logoImg.naturalHeight*(lw/logoImg.naturalWidth));
        coverLogo.push(new Paragraph({alignment:AlignmentType.CENTER,spacing:{before:600,after:300},children:[new ImageRun({data:dataUrlToUint8Array(logoEl.src),transformation:{width:lw,height:lh}})]}));
      }
    }catch(err){ console.warn("Cover logo skipped",err); }
    const coverLabel=text=>new Paragraph({spacing:{before:220,after:80},children:[new TextRun({text,bold:true,allCaps:true,color:NAVY,size:20,font:FONT})]});
    const coverTable=rows=>new Table({
      width:{size:9360,type:WidthType.DXA},
      borders:{top:{style:BorderStyle.SINGLE,color:"BFBFBF",size:4},bottom:{style:BorderStyle.SINGLE,color:"BFBFBF",size:4},insideHorizontal:{style:BorderStyle.SINGLE,color:"BFBFBF",size:2},left:{style:BorderStyle.NONE},right:{style:BorderStyle.NONE},insideVertical:{style:BorderStyle.NONE}},
      rows
    });
    const children=[
      ...coverLogo,
      new Paragraph({alignment:AlignmentType.CENTER,spacing:{after:40},children:[new TextRun({text:"Site Survey Report",bold:true,allCaps:true,color:NAVY,size:44,font:FONT})]}),
      new Paragraph({alignment:AlignmentType.CENTER,spacing:{after:260},children:[run("DFR Deployment Site Assessment",{color:GRAY,size:20})]}),
      coverLabel("Customer"),
      coverTable([
        infoRow("AGENCY",dash(info.agency)),
        infoRow("POINT OF CONTACT",dash([info.pocName,info.pocTitle].filter(Boolean).join(", "))),
        infoRow("PHONE",dash(info.pocPhone)),
        infoRow("EMAIL",dash(info.pocEmail)),
        infoRow("SITE ADDRESS",dash(info.siteAddress)),
        infoRow("PROJECT / JOB #",dash(info.jobNumber)),
        infoRow("SURVEY DATE",effectiveSurveyDate(photos))
      ]),
      coverLabel("Prepared By"),
      coverTable([
        infoRow("NAME",dash([info.installerName,info.installerTitle].filter(Boolean).join(", "))),
        infoRow("PHONE",dash(info.installerPhone)),
        infoRow("EMAIL",dash(info.installerEmail)),
        infoRow("COMPANY","BRINC Drones, Inc."),
        infoRow("GENERATED",new Date().toLocaleString())
      ]),
      new Paragraph({alignment:AlignmentType.CENTER,spacing:{before:300},children:[run(`Revision ${info.revision}  ·  ${info.status}`,{bold:true,color:GRAY,size:20})]}),
      new Paragraph({children:[new PageBreak()]})
    ];
```

Note: `agencyName` (line 2927) stays — the filename logic still uses it.

- [ ] **Step 3: Verify**

Reload, annotate one photo, Download Word Document. Open .docx: page 1 = logo, title, Customer table (empty fields show "—"), Prepared By table, "Revision A · Draft" line. Body starts page 2. Console clean.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add engineering cover page to Word export"
```

---

### Task 4: Word export — bookmarks + static linked TOC

**Files:**
- Modify: `index.html` — `buildWordDocumentBlob`: `sectionHeading` helper (line 2924), section loop (~line 2946), children after cover

**Interfaces:**
- Consumes: cover `children` array from Task 3; `sorted` (existing).
- Produces: bookmark anchors named `loc0`, `loc1`, … matching TOC entries. `sectionHeading(text, anchor)` new signature — the loop call site is updated in this same task.

- [ ] **Step 1: Add classes to destructure**

Line 2917 destructure, add `InternalHyperlink,Bookmark`:
```js
    const {Document,Packer,Paragraph,TextRun,ImageRun,AlignmentType,ExternalHyperlink,Table,TableRow,TableCell,WidthType,BorderStyle,Footer,PageNumber,PageBreak,InternalHyperlink,Bookmark}=window.docx;
```

- [ ] **Step 2: Give section headings bookmark anchors**

Replace the `sectionHeading` helper (line 2924):
```js
    const sectionHeading=text=>new Paragraph({spacing:{before:280,after:140},border:{bottom:{style:BorderStyle.SINGLE,color:NAVY,size:12,space:2}},children:[new TextRun({text,bold:true,allCaps:true,color:NAVY,size:26,font:FONT})]});
```
with:
```js
    const sectionHeading=(text,anchor)=>new Paragraph({spacing:{before:280,after:140},border:{bottom:{style:BorderStyle.SINGLE,color:NAVY,size:12,space:2}},children:[new Bookmark({id:anchor,children:[new TextRun({text,bold:true,allCaps:true,color:NAVY,size:26,font:FONT})]})]});
```

In the location loop (~line 2946), change:
```js
      children.push(sectionHeading(g.name || "Unknown Location"));
```
to:
```js
      children.push(sectionHeading(g.name || "Unknown Location",`loc${gi}`));
```

- [ ] **Step 3: Insert TOC page after cover**

Immediately after the `children` initializer from Task 3 (after the `PageBreak` paragraph), add:

```js
    children.push(new Paragraph({spacing:{after:160},children:[new TextRun({text:"Contents",bold:true,allCaps:true,color:NAVY,size:30,font:FONT})]}));
    sorted.forEach((g,gi)=>{
      children.push(new Paragraph({spacing:{after:90},children:[new InternalHyperlink({anchor:`loc${gi}`,children:[new TextRun({text:`${gi+1}.  ${g.name || "Unknown Location"}`,style:"Hyperlink",font:FONT,size:21,color:LINK,underline:{}})]})]}));
    });
    children.push(new Paragraph({children:[new PageBreak()]}));
```

- [ ] **Step 4: Verify**

Export .docx with photos from 2+ locations (or 1 — TOC then has one entry). Open in Word: page 2 = "CONTENTS" with numbered blue links; Ctrl+Click a link → jumps to that location section. If iPhone available: AirDrop/email the file, confirm links render (not blank) in Word iOS / Quick Look.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add bookmark-linked static TOC to Word export"
```

---

### Task 5: Word export — headers/footers + figure captions

**Files:**
- Modify: `index.html` — `buildWordDocumentBlob`: figure caption line (~line 2957), footer block (line 2980), `sections` config (line 2986)

**Interfaces:**
- Consumes: `info`, `dash` (Task 3); `photo.caption` (Task 2); `g` (group in loop scope).
- Produces: final document layout. Nothing downstream.

- [ ] **Step 1: Add Header and tab classes to destructure**

Line 2917 destructure, add `Header,TabStopType`:
```js
    const {Document,Packer,Paragraph,TextRun,ImageRun,AlignmentType,ExternalHyperlink,Table,TableRow,TableCell,WidthType,BorderStyle,Footer,PageNumber,PageBreak,InternalHyperlink,Bookmark,Header,TabStopType}=window.docx;
```

- [ ] **Step 2: Figure captions use photo caption with location fallback**

In the image loop, replace (~line 2957):
```js
        children.push(new Paragraph({alignment:AlignmentType.CENTER,spacing:{after:100},children:[run(`Figure ${idx}. ${p.fileName}`,{italics:true,color:GRAY,size:18})]}));
```
with:
```js
        const figureText = p.caption ? `Figure ${idx}. ${p.caption}` : `Figure ${idx} — ${g.name || "Unknown Location"}`;
        children.push(new Paragraph({alignment:AlignmentType.CENTER,spacing:{after:100},children:[run(figureText,{italics:true,color:GRAY,size:18})]}));
```

- [ ] **Step 3: Replace footer, add header, wire titlePage**

Replace the footer line (2980):
```js
    const footer=Footer && PageNumber ? new Footer({children:[new Paragraph({alignment:AlignmentType.CENTER,children:[new TextRun({text:"Page ",font:FONT,color:"808080",size:16}),new TextRun({children:[PageNumber.CURRENT],font:FONT,color:"808080",size:16})]})]}) : null;
```
with:
```js
    const footRun=(text)=>new TextRun({text,font:FONT,color:"808080",size:16});
    const footer=Footer && PageNumber ? new Footer({children:[new Paragraph({alignment:AlignmentType.CENTER,children:[footRun(`${info.status}  ·  Rev ${info.revision}  ·  Page `),new TextRun({children:[PageNumber.CURRENT],font:FONT,color:"808080",size:16}),footRun(" of "),new TextRun({children:[PageNumber.TOTAL_PAGES],font:FONT,color:"808080",size:16})]})]}) : null;
    const headerLeft=[info.agency,info.jobNumber?`Job #${info.jobNumber}`:""].filter(Boolean).join("  ·  ") || "Site Survey Report";
    const header=Header ? new Header({children:[new Paragraph({tabStops:[{type:TabStopType.RIGHT,position:9360}],border:{bottom:{style:BorderStyle.SINGLE,color:"BFBFBF",size:4,space:2}},children:[new TextRun({text:headerLeft,font:FONT,color:"808080",size:16}),new TextRun({text:"\tBRINC",font:FONT,color:"808080",size:16,bold:true})]})]}) : null;
    const emptyHeader=Header ? new Header({children:[new Paragraph({children:[]})]}) : null;
    const emptyFooter=Footer ? new Footer({children:[new Paragraph({children:[]})]}) : null;
```

Replace the `sections` line inside `new Document({...})` (line 2986):
```js
      sections:[{properties:{page:{margin:{top:1440,right:1440,bottom:1440,left:1440}}},...(footer?{footers:{default:footer}}:{}),children}]
```
with:
```js
      sections:[{properties:{page:{margin:{top:1440,right:1440,bottom:1440,left:1440}},titlePage:true},...(header?{headers:{default:header,first:emptyHeader}}:{}),...(footer?{footers:{default:footer,first:emptyFooter}}:{}),children}]
```
(`titlePage:true` gives the cover its own blank first-page header/footer.)

- [ ] **Step 4: Verify**

Export .docx. Cover page: no header/footer. Page 2+: header "Agency · Job #… | BRINC" with rule line, footer "Draft · Rev A · Page 2 of N". Captioned photo shows "Figure 1. <caption>"; uncaptioned shows "Figure 2 — <location name>". Export again with ALL project fields empty: cover tables show "—", header falls back to "Site Survey Report", no crash.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add report headers/footers and caption-based figure lines"
```

---

### Task 6: Full manual verification pass

**Files:**
- Modify: none (fix regressions in `index.html` if found)

- [ ] **Step 1: Run the spec's manual test matrix**

1. Fresh load, fill all Project Info fields → export Word → cover/TOC/header/footer all populated.
2. Clear all fields ("New project" + clear BRINC manually) → export → "—" fallbacks everywhere, header shows "Site Survey Report", no console errors.
3. Reload page → BRINC fields persist (localStorage), customer fields persist until "New project".
4. Captions: set on one photo, leave another blank → Word figure lines correct; ZIP `Location_Summary.txt` carries caption.
5. TOC links jump in desktop Word. iPhone if available: file renders, links tappable, no blank TOC.
6. ZIP filename still uses agency from panel (`Agency_<timestamp>.zip`).

- [ ] **Step 2: Fix anything found, commit fixes**

```bash
git add index.html
git commit -m "fix: address issues found in report verification pass"
```
(Skip commit if nothing found.)

- [ ] **Step 3: Update knowledge graph**

Run: `graphify update .` (AST-only). Skip without failing if the `graphify` CLI is not installed.
