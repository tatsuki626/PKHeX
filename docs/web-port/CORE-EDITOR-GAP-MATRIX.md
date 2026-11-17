# Core editor and nested-window gap matrix

Audit date: 2026-11-17  
PKHeX source baseline: `9a1a8f4a450996e9572ee4428e9ae709166c2c6d`  
Host branch inspected: `experiment/upstream-winforms-web-port`  
Scope: Main, SAVEditor, PKMEditor, their launchable child windows, and the nested SAV/PKM dialogs reached from those windows.

This is a source-to-host static audit. A matching endpoint or panel is a **candidate**, not parity proof. Follow-up hardening was committed on the host branch; its local CI verification is pending.

## Status vocabulary

- **Candidate** — a plausible browser panel or API exists, but field coverage, generation gating, validation, commit/cancel and persistence have not been proven.
- **Partial** — a related web workflow exists, but a known scope or semantic difference remains.
- **Missing** — no corresponding browser workflow/API was found in the inspected host source.
- **Excluded** — desktop lifecycle behaviour has no meaningful browser equivalent.
- **Needs trace** — source reachability is known, but the exact child behaviour/host mapping still requires control-level inspection.

## Executive gap matrix

| Editor surface | Desktop scope | Browser counterpart found | Current assessment | Main gaps |
|---|---|---|---|---|
| Main | File/open/save/export, box operations, database, Showdown, QR, report, settings, legality, undo/redo | Save/file APIs, source browse/open, DB search, settings, history, box tools, legality; Showdown routes in registered Phase13Tools module; QR/report panels | **Partial** | Showdown batch routes and separate current-editor preview/apply/export now exist; current-editor draft fields are included and export is non-mutating. New editor integration awaits local CI; custom per-context desktop export preferences remain a gap. QR payload/bitmap and report-grid semantics unverified; file semantics, unsaved guards and batch transaction need tests |
| SAVEditor core | Box/party/extra slots, daycare, search/sort/pop-out, import/export, checksum and stored-entity verification | Boxes/party/battlebox/daycare, slot move, layout, box tools, export/import, trainer/inventory/mail/records/gifts/events/Pokédex | **Partial** | Checksum verification, BulkAnalysis, SAV4BR slot selector and daycare RNG seed missing; save pre-write cleanup and several slot/view behaviours unverified |
| SAVEditor generation-specific children | 84 directly constructed forms plus two nested forms | Generic panels for trainer, inventory, gifts, events, mail, records, Pokédex and raids; some box/layout routes | **Mostly partial or missing** | Most specialist Gen1–Gen9 editors have no one-to-one panel/route identified; generic routes do not establish equivalent field coverage |
| PKMEditor core | Format-aware fields, dependent controls, validation, helper actions, preview and legality | Schema GET/PATCH, action routes, legality; schema-driven tabs | **Partial** | Cross-field validation/focus, dependency refresh, generation-specific field semantics, move flags/PP, advanced metadata editability and format matrix |
| PKMEditor nested dialogs | Ribbon, memory/Amie, Super Training, move shop, tech records, plus records, trash/text, ball/status browsers | Some helper actions and schema fields; no proven complete modal counterparts | **Partial / missing** | Exact dialog semantics, apply/cancel isolation and all format-specific interfaces require explicit coverage |

## Confirmed cross-editor gaps

| ID | Surface | Gap | Evidence / current host state | Priority |
|---|---|---|---|---|
| G-001 | Main / Showdown | Desktop interaction and export-setting parity | Registered `Phase13Tools.Map` retains batch-to-box routes; new Program.cs routes provide current-editor first-set preview/apply and non-mutating export from unsaved form/schema fields, using PKHeX Showdown serialization with configured language/context. Current-editor routes await local CI; custom per-context desktop export preferences remain unavailable | P1 |
| G-002 | Main / QR | End-to-end QR parity | QR panel exists, but inspected flow describes PK7 payload/text output rather than proving bitmap rendering, import and payload compatibility | P1 |
| G-003 | Main / report | Report grid parity | No dedicated report-grid route found; save data alone does not prove desktop columns, extra properties, filters or export | P1 |
| G-004 | Main / file lifecycle | OpenQuick and save semantics | Upload/download exist; standalone PKM save, multiple-file handling, unsaved PKM warning, export filename and browser close warning remain partial | P1 |
| G-005 | Main / batch | Transaction semantics | Box-tools preview/apply exists; desktop BatchEditor only commits on OK. Cancel, partial validation and undo integration need proof | P1 |
| G-006 | SAVEditor / checksum | Dedicated checksum verification | Implemented in host as GET `/saves/{id}/verification/checksum` with durable edited-state guard and `SaveFile.ChecksumInfo` diagnostics; browser exposes verify/copy workflow | **Implemented; local CI passed (2026-11-17)** |
| G-007 | SAVEditor / entity audit | Whole-save stored-entity analysis | Implemented in host as GET `/saves/{id}/verification/stored-entities` using `PKHeX.Core.Bulk.BulkAnalysis`, localized report, entity/finding/clone counts; browser exposes report/copy workflow | **Implemented; local CI passed (2026-11-17)** |
| G-008 | SAVEditor / daycare | RNG seed editing | Daycare state route covers experience/egg state; no typed UInt16/32/64/128 seed field or write route | P1 |
| G-009 | SAVEditor / SAV4BR | Current save-slot selection | No selector/route found; desktop closes BattlePass and resets box names/viewers on switch | P1 |
| G-010 | SAVEditor / save export | Pre-write validation/cleanup | Download route exists; desktop `ValidateChildren()` and `IStorageCleanup.FixStoragePreWrite()` semantics are not established | P1 |
| G-011 | SAVEditor / generation children | Specialist save editors | Most child forms have no corresponding route or panel; generic save blocks/data endpoints do not prove equivalent editing | P1 |
| G-012 | PKMEditor / validation | Cross-field validation and invalid-field navigation | PATCH/schema exist; equivalent to `PreparePKM`, `GetInvalidParentTab`, `IsNotValid` not proven | P0 |
| G-013 | PKMEditor / dependencies | Species/form/origin/ability/location refresh | Schema option generation exists; dependent refresh and stale-value handling need tests | P1 |
| G-014 | PKMEditor / moves | PP, relearn, alpha/mastery, move-shop and flags | Schema move fields/actions exist; detailed format-specific semantics unverified | P1 |
| G-015 | PKMEditor / format rules | PID/EC/shiny/gender/nature and met/egg invariants | Helper actions exist; generation-specific constraints and read-after-write tests needed | P1 |
| G-016 | PKMEditor / advanced | Advanced metadata editability | Advanced schema is partly read-only; each field needs explicit editable/read-only classification | P1 |
| G-017 | Nested editors | Commit/cancel isolation | Browser panels/actions do not yet demonstrate modal working-copy semantics across nested editors | P1 |
| G-018 | Shared history | Undo/redo coverage | Host history endpoints exist; every nested/multi-slot mutation must be checked for a single coherent undo unit | P1 |

## SAVEditor child-window inventory

Launch dispatch was inspected in `PKHeX.WinForms/Controls/SAV Editor/SAVEditor.cs` (notably lines 626–913 and 1438–1479), plus the two nested launch sites recorded in `inventory.csv`. There are **84 distinct SAV_* forms directly constructed by SAVEditor** and **two additional nested forms** reached from Pokédex windows. Reused forms such as the three raid-origin variants count once by form type.

### Child windows with an identifiable browser feature family

| Desktop form(s) | Desktop entry | Browser family found | Assessment / remaining proof |
|---|---|---|---|
| `SAV_Inventory` | Item pouch button | `GET /saves/{id}/inventory`, `PUT /inventory/{pouchName}`; Inventory panel | **Candidate** — compare pouch availability, item limits, sorting, quantities, empty slots and generation-specific rules |
| `SAV_SimpleTrainer`, `SAV_Trainer`, `SAV_Trainer7`, `SAV_Trainer7GG`, `SAV_Trainer8`, `SAV_Trainer8b`, `SAV_Trainer8a`, `SAV_Trainer9`, `SAV_Trainer9a`, `SAV_Trainer4BR` | Trainer info dispatch by concrete save type | `GET/PATCH /saves/{id}/trainer`; Trainer panel | **Partial** — one generic model is not proof of every generation's trainer fields, badges, play time, IDs, language, avatar and special flags; notably SAV4BR requires its own slot context |
| `SAV_SimplePokedex`, `SAV_Pokedex4`, `SAV_Pokedex5`, `SAV_PokedexXY`, `SAV_PokedexORAS`, `SAV_PokedexSM`, `SAV_PokedexGG`, `SAV_PokedexSWSH`, `SAV_PokedexBDSP`, `SAV_PokedexLA`, `SAV_PokedexSV`, `SAV_PokedexSVKitakami`, `SAV_Pokedex9a` | Pokédex dispatch by save type/revision | `GET/PATCH /saves/{id}/pokedex`, bulk preview/apply; Pokédex panel | **Partial** — verify per-generation caught/seen/form/gender/language flags, regional dex distinctions, Kitakami/revision selection and bulk semantics |
| `SAV_EventReset1`, `SAV_EventWork`, `SAV_FlagWork8b`, `SAV_EventFlags`, `SAV_EventFlags2`, `SAV_FlagWork9a` | Event button switch by save type/interface | `GET /events/flags`, preview/apply; Events panel | **Partial** — inspect each generation's flag/work width, named ranges, reset semantics and advanced/system flags |
| `SAV_Wondercard` | Wondercard button | Gifts routes and Gifts panel | **Partial** — compare card slot selection, import/export formats, wonder-news/card metadata, injection and replacement confirmations |
| `SAV_GiftRibbons` | Gift-ribbon editor button | Ribbon fields/actions in PKMEditor schema are not equivalent to save-level gift-ribbon storage; no save-level route identified | **Missing** — save-level `IGiftRibbons` collection editor, generation gating and persistence |
| `SAV_Raid8`, `SAV_Raid9`, `SAV_RaidSevenStar9` | Raid button variants | No raid-specific route found in inspected Program.cs; generic save data routes are not a raid editor | **Missing / needs implementation** — distinct Galar/Isle of Armor/Crown Tundra and Paldea/Kitakami/Blueberry/Seven-Star origin handling |
| `SAV_BoxLayout` | Box layout button | `GET/PUT /boxes/{box}/layout` | **Candidate** — verify wallpaper/background/name ordering, restrictions and refresh of all open viewers |
| `SAV_MailBox` | Mailbox button | `GET/PATCH/DELETE /mail`; Mail panel | **Partial** — compare mailbox vs mail-slot distinctions, party refresh, word tables, invalid mail and all supported save generations |
| `SAV_GroupViewer` | Stadium registered-team viewer | No matching registered-team group route/panel found | **Missing** — multi-team slot group editing and PKMEditor integration |
| `SAV_BoxViewer`, `SAV_BoxList` | Box double-click / Shift-double-click | Boxes panel, box/slot APIs, pop-out/search frontend | **Partial** — verify independent viewer lifecycle, multi-view synchronization, drag/drop, context menu, sorting, search and list navigation |
| `SAV_BattlePass` | SAV4BR Battle Pass button | No matching endpoint/panel found | **Missing** — SAV4BR Battle Pass data editor; also must close/reset when current save slot changes |
| `SAV_BlockDump8` | Save-block button for `ISCBlockArray` | Generic data and save APIs exist, but no block-dump editor contract found | **Needs trace / likely missing** — enumerate block keys, raw export/import, size validation and write permissions |
| `SAV_Fashion9` | Fashion button | Host has fashion-related data capability, but no dedicated Fashion panel/route was identified in the inspected route/UI set | **Partial / needs trace** — prove actual clothing/appearance catalog, unlock flags and save mutation coverage |
| `SAV_RTC3` | Gen3 RTC button | No RTC editor route/panel found | **Missing** — date/time/offset and reset-state editing with generation-specific validation |

### Generation-specific specialist forms without a one-to-one browser counterpart identified

These are confirmed reachable launch targets, not orphan candidates. For each one, the next action is to inspect its own fields and backing PKHeX.Core interfaces, then add a typed route/panel or document a deliberate exclusion. Do not silently map these to generic inventory/data endpoints.

| Generation / family | Reachable form types | Current assessment |
|---|---|---|
| Gen1 | `SAV_HallOfFame1` | **Missing** — Hall of Fame read/edit/export semantics |
| Gen2 | `SAV_Misc2`, `SAV_EventFlags2` (event workflow above), RTC reset action | **Missing / partial** — miscellaneous flags/state; RTC reset currently only prompt-driven Core action |
| Gen3 | `SAV_HallOfFame3`, `SAV_Misc3`, `SAV_Roamer3`, `SAV_SecretBase3` | **Missing** — Hall of Fame, miscellaneous state, roaming encounter state, secret-base data |
| Gen4 | `SAV_Apricorn`, `SAV_Pokeathlon4`, `SAV_Safari4`, `SAV_HoneyTree`, `SAV_Geonet4`, `SAV_DLC4`, `SAV_Gear`, `SAV_Misc4`, `SAV_Trainer4BR`, `SAV_Underground` | **Missing / partial** — apricorns, Pokéathlon records, Safari Zone, honey trees, Geonet, DLC flags, Battle Revolution gear, misc fields, Underground; trainer is covered only by generic trainer route |
| Gen5 | `SAV_UnityTower`, `SAV_JoinAvenue`, `SAV_Medals5`, `SAV_GlobalLink5`, `SAV_DLC5`, `SAV_Misc5` | **Missing** — Unity Tower, Join Avenue, medals, Global Link, DLC and miscellaneous state |
| Gen6 | `SAV_BerryFieldXY`, `SAV_PokeBlockORAS`, `SAV_SuperTrain`, `SAV_Link6`, `SAV_OPower`, `SAV_SecretBase`, `SAV_Roamer6`, `SAV_Pokepuff`, `SAV_HallOfFame` | **Missing** — berries, Pokéblocks, Super Training, link data, O-Powers, Secret Bases, roamer, Poké Puffs and Hall of Fame |
| Gen7 | `SAV_Pokebean`, `SAV_ZygardeCell`, `SAV_Chatter`, `SAV_FestivalPlaza`, `SAV_HallOfFame7` | **Missing** — beans, Zygarde cells, chatter, Festival Plaza and Hall of Fame |
| Gen8 | `SAV_Underground8b`, `SAV_SealStickers8b`, `SAV_Poffin8b`, `SAV_Misc8b` | **Missing** — Underground, stickers, Poffins and miscellaneous BDSP state |
| Gen9 / ZA | `SAV_Donut9a` | **Missing** — donut/ZA-specific save state |

### Additional nested SAV windows

| Nested form | Launch chain | Assessment |
|---|---|---|
| `SAV_Capture7GG` | `SAVEditor` dispatch → `SAV_PokedexGG` → capture dialog | Reachable nested dialog, not orphan. **Missing / needs trace** — captured/caught transfer counts and aggregate Pokédex totals need a specific browser workflow or explicit integration into the Gen7 Pokédex panel |
| `SAV_PokedexResearchEditorLA` | `SAVEditor` dispatch → `SAV_PokedexLA` → research editor | Reachable nested dialog. **Missing / needs trace** — research task progress, level/bonus thresholds and completion semantics must be represented in the LA Pokédex panel |

### Shared SAVEditor operations that are not child forms

- Daycare: current browser covers slots, experience and egg availability. Add daycare-group switching and typed RNG seed read/write; preserve save capability gates and save-edited state.
- Box operations: compare desktop clone-to-box, binary export, load/dump all boxes, import override choices, folder import, locked-slot skips, sorting, search, drag/drop and multi-view synchronization.
- Slot context menu: trace every `ContextMenuSAV` command (open/edit/clone/delete/transfer/legality/file operations) against the browser slot context menu.
- Save verification: add checksum diagnostics and whole-save stored-entity `BulkAnalysis` as separate operations; neither is a substitute for legality checks.
- SAV4BR current slot: implement selection and ensure stale BattlePass/subview instances are invalidated and all box names/slots are refreshed.

## PKMEditor nested-window inventory

The inventory records the following direct PKMEditor child dialogs. They are all reachable from the PKMEditor; the web schema/action surface is not proof of full modal parity.

| Child window | Desktop responsibility | Browser counterpart found | Assessment |
|---|---|---|---|
| `RibbonEditor` | Ribbon/mark/medal editing | Ribbon fields/actions in schema | **Partial** — full collection, format gating, bulk operations and apply/cancel semantics |
| `MemoryAmie` | Memory/Amie affection and related fields | Some metadata fields in schema | **Missing / partial** — dedicated memory options, generation-specific state and localized text |
| `SuperTrainingEditor` | Gen6 Super Training state | No dedicated PKM action/panel confirmed | **Missing** — separate from SAV-level Super Training editor |
| `MoveShopEditor` | Legends/format-specific move-shop state | Move fields and helper actions | **Partial** — shop unlock/learned flags, mastery and current/relearn state are distinct |
| `TechRecordEditor` | Technical record flags | Move/technical-record schema fields may exist | **Partial** — verify per-format TR compatibility and exact bitset persistence |
| `PlusRecordEditor` | Plus-record values | No dedicated editor confirmed | **Missing / needs trace** — inspect which formats expose the values and their limits |
| `TrashEditor` | Raw string/trash bytes and encoding | Text fields in schema | **Partial** — byte-level encoding, terminators, language and preservation need tests |
| `BallBrowser` | Ball selection browser | Ball schema option field | **Candidate** — compare sprite/catalog, legality filtering and unsupported legacy ball behaviour |
| `StatusBrowser` | Status condition selector | Status fields/action in schema | **Candidate** — compare status enum, format gates and special conditions |

Additional reusable PKMEditor controls requiring explicit owner/mapping rather than new windows: `CatchRate`, `ContestStat`, `EntityInstructionBuilder`, `FormArgumentEditor`, `MoveChoice`, `ShinyLeaf`, `SizeCP`, `StatEditor`, `StatusConditionView`, `TrainerID`, `TrainerSID`, `TrainerTID`, `ExperienceBar`, `GenderToggle`, and `ContextMenuPKM`.

## Main child-window / callback dependencies

Main's child callbacks cross editor boundaries and must remain in the parity matrix:
- `SAVEditor.RequestCloneData` → Main clone workflow.
- `SAVEditor.RequestReloadSave` → Main save/source reload.
- `PKMEditor.LegalityChanged` and `UpdatePreviewSprite` → Main legality/preview state.
- `PKMEditor.RequestShowdownImport` / `RequestShowdownExport` → Main Showdown workflow (currently backend route gap G-001).
- `PKMEditor.SaveFileRequested` → standalone PKM save semantics (G-004).

## Implementation order

1. **P0 correctness:** G-012 PKMEditor validation contract; G-001 Showdown editor-interaction/export-setting parity. G-006 checksum diagnostics and G-007 stored-entity BulkAnalysis are implemented and passed the complete local CI suite on 2026-11-17.
2. **P1 core parity:** SAV daycare RNG and SAV4BR slot; save pre-write cleanup; PKM dependent-field/move/format tests; Main QR/report/file lifecycle.
3. **P1 generation children:** implement typed route families, beginning with Hall of Fame/misc/RTC/roamer/secret base and Gen4–Gen9 specialist state. Group only where the underlying Core interface and field semantics genuinely match.
4. **Nested editors:** PKM nested dialog equivalents and SAV Pokédex nested research/capture dialogs; define apply/cancel and undo unit boundaries.
5. **Verification:** representative real save fixtures for each supported generation; targeted read-after-write, invalid-input, cancel, undo/redo, export byte-equivalence and capability-gating tests.

## Source evidence

- `PKHeX.WinForms/Controls/SAV Editor/SAVEditor.cs` — SAVEditor dispatch, child construction, daycare, box and shared operations.
- `PKHeX.WinForms/Controls/SAV Editor/SAVEditor.Designer.cs` — SAVEditor control/event inventory.
- `PKHeX.WinForms/Controls/PKM Editor/PKMEditor.cs` and designer — PKMEditor handlers and nested dialog launches.
- `docs/web-port/MAIN-HANDLER-VERIFICATION.md`
- `docs/web-port/SAVEDitor-HANDLER-VERIFICATION.md`
- `docs/web-port/PKMEditor-HANDLER-VERIFICATION.md`
- Host `Program.cs` and `wwwroot/js/app.js` on `experiment/upstream-winforms-web-port`.

Static-audit limitation: each child form's own designer and implementation still need field-by-field inspection before claiming complete parity. The launch inventory is complete for SAVEditor dispatch in the audited source, but this document intentionally keeps the specialist forms in **missing/needs-trace** until their own controls and Core interfaces are audited.


### Verification update — 2026-11-17

- Main current-PKM Showdown preview/apply/export: **local CI verified**. The active-editor endpoints, draft-aware UI, schema-field filtering, stale-preview protection, no-mutation export, and undoable apply passed the complete local CI suite at `2026-11-17T10:08:56+04:00`.
- Remaining limitation: web settings do not yet expose the desktop's custom per-context Showdown export preferences. Treat this as a parity follow-up, not a CI blocker.
- Main QR, report-grid and encounter-trainer-database flows remain partial pending end-to-end tracing. PKMEditor cross-field/dependent-field validation remains a high-priority gap.
