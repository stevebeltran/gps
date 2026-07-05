# Graph Report - gps  (2026-07-03)

## Corpus Check
- 16 files · ~78,885 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 95 nodes · 80 edges · 15 communities (9 shown, 6 thin omitted)
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `f31e4cf7`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- [[_COMMUNITY_Community 0|Community 0]]
- [[_COMMUNITY_Community 1|Community 1]]
- [[_COMMUNITY_Community 2|Community 2]]
- [[_COMMUNITY_Community 3|Community 3]]
- [[_COMMUNITY_Community 4|Community 4]]
- [[_COMMUNITY_Community 5|Community 5]]
- [[_COMMUNITY_Community 6|Community 6]]
- [[_COMMUNITY_Community 7|Community 7]]
- [[_COMMUNITY_Community 8|Community 8]]
- [[_COMMUNITY_Community 9|Community 9]]
- [[_COMMUNITY_Community 10|Community 10]]
- [[_COMMUNITY_Community 11|Community 11]]
- [[_COMMUNITY_Community 12|Community 12]]
- [[_COMMUNITY_Community 13|Community 13]]
- [[_COMMUNITY_Community 14|Community 14]]

## God Nodes (most connected - your core abstractions)
1. `Annotation Import: survey drawing tools → gps app` - 10 edges
2. `Professional Survey Report Upgrade — Design` - 8 edges
3. `Verification harness (used by every task)` - 7 edges
4. `What I did` - 6 edges
5. `Global Constraints` - 6 edges
6. `Task 1 Report: Project Info Panel with localStorage Persistence` - 5 edges
7. `Task 2 Implementation Report: Photo captions` - 5 edges
8. `Task 3 Completion Report: Word Export Cover Page` - 5 edges
9. `Task 4 Report: Word Export Bookmarks + Static TOC` - 5 edges
10. `Task 5 Completion Report: Word export — headers/footers + figure captions` - 5 edges

## Surprising Connections (you probably didn't know these)
- None detected - all connections are within the same source files.

## Import Cycles
- None detected.

## Communities (15 total, 6 thin omitted)

### Community 0 - "Community 0"
Cohesion: 0.14
Nodes (13): Concerns, Deviations from Brief, Diff Review, Git Commit, Grep Verification, JavaScript Parse Check, Step 1: Destructure Header and TabStopType, Step 2: Figure Captions with Photo Caption + Location Fallback (+5 more)

### Community 1 - "Community 1"
Cohesion: 0.18
Nodes (10): Concerns, Deviations from brief, Step 1: Added panel HTML (after line 612), Step 2: Removed old agency input, Step 3: Added panel CSS, Step 4: Added project-info JS (before setStatus function), Step 5: Replaced agency input references, Task 1 Report: Project Info Panel with localStorage Persistence (+2 more)

### Community 2 - "Community 2"
Cohesion: 0.18
Nodes (10): Annotation Import: survey drawing tools → gps app, Approach (chosen: extend existing system), Behavior, Data model, Goal, Out of scope (YAGNI, matches survey behavior), Rendering and export, Source material (survey app) (+2 more)

### Community 3 - "Community 3"
Cohesion: 0.20
Nodes (9): Global Constraints, Professional Survey Report Upgrade Implementation Plan, Task 1: Project Info panel (UI + localStorage), Task 2: Photo captions, Task 3: Word export — cover page, Task 4: Word export — bookmarks + static linked TOC, Task 5: Word export — headers/footers + figure captions, Task 6: Full manual verification pass (+1 more)

### Community 4 - "Community 4"
Cohesion: 0.22
Nodes (8): 1. Project Info panel (UI), 2. Photo captions, 3. Word document rebuild (`buildWordDocumentBlob`), 4. Error handling, 5. Testing (manual), Out of scope, Professional Survey Report Upgrade — Design, Purpose

### Community 5 - "Community 5"
Cohesion: 0.25
Nodes (7): Annotation Import (survey → gps) Implementation Plan, Global Constraints, Task 1: Data model, symbol tables, drawing primitives, Task 2: Toolbar UI (HTML + CSS), Task 3: Tool selection, pointer handlers, unified undo, Task 4: Rendering, exports, annotated-state checks, Task 5: End-to-end manual verification

### Community 6 - "Community 6"
Cohesion: 0.29
Nodes (6): Changes made (index.html only):, Concerns, Deviations from brief, Task 2 Implementation Report: Photo captions, Verification output, What I did

### Community 7 - "Community 7"
Cohesion: 0.29
Nodes (6): Changes Made, Concerns, Deviations from Brief, Task 4 Report: Word Export Bookmarks + Static TOC, Verification Output, What I Did

### Community 8 - "Community 8"
Cohesion: 0.33
Nodes (5): Concerns, Deviations from Brief, Task 3 Completion Report: Word Export Cover Page, Verification Output, What I Did

## Knowledge Gaps
- **64 isolated node(s):** `SDD progress ledger — professional-report`, `Task 1: Project Info panel (UI + localStorage)`, `Step 1: Added panel HTML (after line 612)`, `Step 2: Removed old agency input`, `Step 3: Added panel CSS` (+59 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **6 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What connects `SDD progress ledger — professional-report`, `Task 1: Project Info panel (UI + localStorage)`, `Step 1: Added panel HTML (after line 612)` to the rest of the system?**
  _64 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Community 0` be split into smaller, more focused modules?**
  _Cohesion score 0.14285714285714285 - nodes in this community are weakly interconnected._