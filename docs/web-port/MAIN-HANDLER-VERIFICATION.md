# Main window — handler-to-web verification

Reference source baseline: `9a1a8f4a450996e9572ee4428e9ae709166c2c6d` (branch `web-port/form-inventory-and-parity`).
Host inspected: `experiment/upstream-winforms-web-port`, current `Program.cs` and `wwwroot/js/app.js`.
Scope: Main menu/event handlers and their concrete browser/API counterparts. This is source inspection, not runtime verification.

## Findings that change the gap assessment

The web frontend contains a Showdown panel and constructs links/requests for:
- `GET /api/saves/{id}/showdown/export?scope=slot|party|box|all`
- `POST /api/saves/{id}/showdown/import/preview`
- `POST /api/saves/{id}/showdown/import/apply`

The inspected ASP.NET route declarations contain **none** of these paths. The panel is therefore not evidence of a working Showdown implementation. Treat all Showdown scopes as **partial / backend missing** until routes are implemented and exercised. The earlier feature-level candidate status was too optimistic.

Likewise, the Main desktop QR command calls QR import/export code in WinForms. The web app has a QR panel, but this pass has not yet proven payload parity, error handling, or a corresponding server operation. Keep it **partial / verification required**, not complete.

## Handler map

| WinForms control / handler | Desktop responsibility | Web counterpart found | Audit result |
|---|---|---|---|
| `Menu_Open → MainMenuOpen` | Open supported save/PKM file through format-aware file dialog and `OpenQuick` | Save upload `POST /api/saves`; source browse/open endpoints; browser file input | **Partial** — upload exists; validate accepted formats, multi-file/open-PKM behaviour and error paths |
| `Menu_Save → MainMenuSave` | Requires editor fields complete, prepares current PKM, saves a PKM file | Slot file download endpoints exist for box/party/battlebox; schema PATCH persists into save | **Partial** — desktop saves current edited PKM as a standalone file; verify exact browser equivalent and unsaved editor state |
| `Menu_ExportSAV → ClickExportSAV` | Checks enabled state; optionally warns if current PKM unsaved; exports current save | `GET /api/saves/{id}/download` | **Partial** — endpoint exists; browser confirmation/unsaved-entity guard and filename semantics need verification |
| `Menu_Exit → MainMenuExit` | Ctrl-triggered confirmation then closes app | No browser equivalent needed | **Excluded** — browser tab lifecycle replaces application exit; do not add a fake exit action |
| `Menu_ShowdownImportPKM → ClickShowdownImportPKM` | Reads clipboard, parses first set, confirms normalized set, reports invalid lines, loads into editor | Showdown panel calls preview/apply endpoints | **Partial / backend missing** — frontend requests routes absent from inspected Program.cs |
| `Menu_ShowdownExportPKM → ClickShowdownExportPKM` | Requires valid editor state, serializes current PKM using language/context settings, writes clipboard and verifies it | Showdown panel/link requests export endpoint | **Partial / backend missing** — endpoint absent; clipboard UX and serialization settings also unverified |
| `Menu_ShowdownExportParty → ClickShowdownExportParty` | Exports party scope | Frontend party export URL | **Partial / backend missing** — endpoint absent |
| `Menu_ShowdownExportCurrentBox → ClickShowdownExportCurrentBox` | Exports current box scope | Frontend current-box export URL | **Partial / backend missing** — endpoint absent |
| `Menu_LoadBoxes → MainMenuBoxLoad` | Optional database directory selection, then loads boxes and reports result | Box import and box data APIs | **Partial** — verify full-database import semantics and user confirmation |
| `Menu_DumpBoxes → MainMenuBoxDump` | Exports all boxes; can dump into database or open exporter dialog | Box export APIs and database catalog | **Partial** — compare all-box scope, override options, file naming and duplicate handling |
| `Menu_DumpBox → MainMenuBoxDumpSingle` | Exports current box | `GET /api/saves/{id}/boxes/{box}/export` | **Candidate** — verify exact format and output behaviour |
| `Menu_Report → MainMenuBoxReport` | Opens report grid populated from all save slots and configured visible/extra properties | Frontend report panel / save and Pokémon data | **Partial** — no dedicated report API route found; verify report columns, filtering, export and empty slots |
| `Menu_Database → MainMenuDatabase` | Opens save-linked database; Shift opens KChart; checks local database directory | `/api/db/status`, `/api/db/search`, reindex and save search | **Partial** — general database exists; save-linked database actions and Shift/KChart parity unverified |
| `Menu_EncDatabase → Menu_EncDatabase_Click` | Recursively scans trainer directory, builds encounter trainer database, opens encounter UI | Frontend encounter panel | **Partial** — verify source discovery, indexing and encounter matching; no dedicated route proven in this pass |
| `Menu_MGDatabase → MainMenuMysteryDB` | Opens Mystery Gift database window | Gift DB and save gift APIs | **Candidate** — verify browse/filter/import semantics and source dataset |
| `Menu_BatchEditor → MainMenuBatchEditor` | Edits prepared current PKM and save slots; commits modified slots only on OK | Box tools preview/apply, bulk actions, Pokémon schema PATCH | **Partial** — verify editor field coverage, transaction/cancel semantics and changelog/undo integration |
| `Menu_Folder → MainMenuFolder` | Lists source folder files and opens selected save clone | `/api/source/browse`, `/api/source/open` | **Candidate** — verify clone isolation, path safety and supported file types |
| `Menu_Undo → ClickUndo` / `Menu_Redo → ClickRedo` | Save-slot changelog undo/redo | `POST /api/saves/{id}/undo`, `/redo`; history endpoint | **Candidate** — verify editor state and save-level history scope match desktop semantics |
| `Menu_Settings → MainMenuSettings` | Edits settings; reloads settings, may replace blank save version and refreshes previews/slots | `GET/PUT /api/settings`; frontend settings controls | **Partial** — browser settings subset may be narrower; startup save version, language and preview effects need comparison |
| `Menu_About → MainMenuAbout` | About/version/license information | `GET /api/about` and About panel | **Candidate** — compare displayed source commit, version and license |
| `PB_Legal → ClickLegality` | Requires completed edits and valid nonempty/checksummed PKM; runs legality analysis using save personal data and opens report | Slot legality endpoint and legality panel | **Partial** — verify same unsaved editor object and save-specific personal data are used |
| `dragout.DragDrop → DragoutDrop` | Imports first dropped file through `OpenQuick`; related drag handlers provide visual feedback | Browser file upload / drag-and-drop | **Partial** — verify drag/drop accepts same file classes and reports failures |
| `Main_FormClosing` | Warns on edited save or unsaved PKM; persists settings | Browser unload lifecycle | **Partial** — browser cannot reliably await async settings save on close; implement best-effort persistence and before-unload warning where supported |

## Important cross-component callbacks

The designer also wires callbacks from child controls:
- `C_SAV.RequestCloneData → ClickClone`
- `C_SAV.RequestReloadSave → ClickSaveFileName`
- `PKME_Tabs.LegalityChanged → PKME_Tabs_LegalityChanged`
- `PKME_Tabs.UpdatePreviewSprite → PKME_Tabs_UpdatePreviewSprite`
- `PKME_Tabs.RequestShowdownImport → PKME_Tabs_RequestShowdownImport`
- `PKME_Tabs.RequestShowdownExport → PKME_Tabs_RequestShowdownExport`
- `PKME_Tabs.SaveFileRequested → PKME_Tabs_SaveFileRequested`

These are part of Main parity even though the initiating controls live inside SAVEditor/PKMEditor. Their browser equivalents must be traced from the child editor actions, not counted as Main menu buttons.

## Verified route inventory relevant to Main

Present in inspected `Program.cs`: save upload/list/download/original, source browse/open, database status/search/reindex, settings GET/PUT, about, undo/redo/history, box exports/import, slot file downloads, legality endpoints, and Pokémon schema/edit/action endpoints.

Not present in inspected `Program.cs`: any Showdown import/export route. No dedicated QR, Main report-grid, or trainer-encounter-database route was identified in this pass. Some frontend panels may perform local-only work; this requires separate end-to-end tracing rather than assuming the route must exist.

## Required follow-up tests

1. Upload representative save and standalone PKM files; compare accepted types and error messages with `OpenQuick`.
2. Edit a PKM without committing it, invoke standalone Save, then export the save; verify prompts and bytes.
3. Exercise Showdown single/party/current-box/all scopes. First implement missing backend routes; assert preview/apply is explicit, only intended slots change, invalid lines are surfaced, and cancel is non-mutating.
4. Exercise box dump/load with duplicates, empty slots and format conversion; compare counts and bytes.
5. Compare report-grid properties and batch editor cancel/commit/undo behaviour.
6. Test drag/drop and unsaved-close warning in supported browsers.

## Status rule

A frontend control or a matching route is only a candidate. Mark **implemented** only after tracing the full action from UI through validation and persistence, and adding a targeted test for the relevant behaviour.
