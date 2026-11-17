# Detailed control mapping: Main, SAVEditor and PKMEditor

## Audit scope and method

- Desktop source: `tatsuki626/PKHeX`, branch `web-port/form-inventory-and-parity`.
- Frozen source baseline for this audit: `9a1a8f4a450996e9572ee4428e9ae709166c2c6d` (2026-11-17).
- Web counterpart: `tatsuki626/pkhex-docker`, branch `experiment/upstream-winforms-web-port`; inspect the current branch when resolving each item.
- Primary source files:
  - `PKHeX.WinForms/MainWindow/Main.Designer.cs`, `Main.cs`
  - `PKHeX.WinForms/Controls/SAV Editor/SAVEditor.Designer.cs`, `SAVEditor.cs`
  - `PKHeX.WinForms/Controls/PKM Editor/PKMEditor.Designer.cs`, `PKMEditor.cs`
- Existing machine inventory: [control-audit.csv](control-audit.csv). It enumerates designer fields and explicit event bindings. This document adds semantic grouping, owning workflow, known web route/component candidates and unresolved parity questions.
- The earlier inventory's 346 designer fields and 216 explicit event bindings are the authoritative mechanically enumerated counts. Dynamic subscriptions, inherited controls and controls created at runtime are reviewed separately below.
- **Status vocabulary:** `candidate` means a counterpart appears to exist but behaviour is not yet verified; `partial` means only part of the desktop workflow is represented; `unmapped` means no reliable counterpart has yet been demonstrated. None of these is parity sign-off.

## Main — composition shell and global commands

### Layout and owned editors

| WinForms control | Role | Web counterpart / assessment |
|---|---|---|
| `C_SAV` | Embedded save editor; owns box, party, daycare and save-specific workflows | Existing save workspace and save navigation. Candidate; map each child workflow in the SAVEditor section. |
| `PKME_Tabs` | Embedded Pokémon editor | Existing schema-driven editor (`renderSchema`) for selected box/party/battle-box/daycare Pokémon. Candidate; schema field and validation parity remains open. |
| `menuStrip1`, `Menu_File`, `Menu_Tools`, `Menu_Options` | Global command/navigation hierarchy | Existing browser toolbar/panels and settings. No one-to-one menu parity demonstrated; each command is mapped below. |
| `splitContainer1`, `splitContainer2` | Desktop split layout | Responsive browser layout; presentation-only, not a domain operation. |
| `dragout` | Drag selected Pokémon out to a file/target; context menu supports legality, QR and Save As | Browser download/export and editor actions exist in part. Desktop OS drag-out and context-menu parity are unverified. |
| `PB_Legal` | Clickable legality indicator and drop target | Existing per-Pokémon legality endpoint/report. Candidate; drag/drop target behaviour is unverified. |
| `L_UpdateAvailable` | Runtime update link, shown only when update check finds a newer version | No matching update-notification workflow identified in the inspected web UI/API. |
| `toolTip`, separators | Hints and visual grouping | Browser title/help text and CSS; presentation-only. |

### File menu

| Control / handler | Desktop behaviour | Web mapping | Assessment |
|---|---|---|---|
| `Menu_Open` → `MainMenuOpen` | Open save or Pokémon file using supported extensions; dispatches through `OpenQuick` | Save upload/list workflow; source browse/open; save and Pokémon import endpoints | Candidate. Verify file type routing, unsupported-file errors and preserving active save/editor state. |
| `Menu_Save` → `MainMenuSave` | Refuses incomplete PKM edits (`EditsComplete`), prepares PKM and opens Save PKM dialog | Pokémon editor PATCH and slot/file export endpoints | Partial. Confirm standalone PKM export, filename/format choice and validation-before-export. |
| `Menu_ExportSAV` → `ClickExportSAV` | Exports active save, with backup/export checks | `GET /api/saves/{id}/download`, original download and backup endpoints | Candidate. Verify native save export, original-vs-edited semantics and backup prompt. |
| `Menu_Exit` → `MainMenuExit` | Quit; Ctrl+Q asks for confirmation | Browser session/app navigation | Not directly portable. Define browser-safe equivalent; do not close the browser or silently discard edits. |

### Tools menu

| Control / handler | Desktop behaviour | Web mapping | Assessment |
|---|---|---|---|
| `Menu_ShowdownImportPKM` → `ClickShowdownImportPKM` | Import Showdown text into editor | Existing Showdown-related frontend capability must be checked against this exact single-PKM flow | Existing Phase 13 Showdown feature; verify this exact current-PKM import flow and invalid-set handling. |
| `Menu_ShowdownExportPKM` → `ClickShowdownExportPKM` | Export current Pokémon as Showdown text | Existing Showdown feature candidate | Existing Phase 13 Showdown feature; verify current-PKM export and browser copy/download semantics. |
| `Menu_ShowdownExportParty` → `ClickShowdownExportParty` | Export party | Existing party view and Showdown candidate | Existing Phase 13 Showdown feature; verify party scope, order and output formatting. |
| `Menu_ShowdownExportCurrentBox` → `ClickShowdownExportCurrentBox` | Export current box | Existing box grid and Showdown candidate | Existing Phase 13 Showdown feature; verify current-box scope, empty slots and output formatting. |
| `Menu_LoadBoxes` → `MainMenuBoxLoad` | Load/import box data | Box import endpoint and box ZIP import UI | Candidate. Verify supported legacy formats and replacement/merge semantics. |
| `Menu_DumpBoxes` → `MainMenuBoxDump` | Export all boxes | `GET /api/saves/{id}/boxes/export` | Candidate. Verify format and complete box coverage. |
| `Menu_DumpBox` → `MainMenuBoxDumpSingle` | Export current box | `GET /api/saves/{id}/boxes/{box}/export` | Candidate. |
| `Menu_Report` → `MainMenuBoxReport` | Opens `ReportGrid`, populates slot cache and configured extra/hidden properties | Existing Phase 13 report-grid/report workflow is listed in the web roadmap; exact column configuration, filters and export parity still need checking | Candidate. Verify report data columns, configured extra/hidden properties, sorting/filtering and export. |
| `Menu_Database` → `MainMenuDatabase` | Opens Pokémon database browser | `/api/db/status`, `/api/db/search`, `/api/db/reindex` and database panel | Candidate. Verify filters, preview, import-to-save and database source handling. |
| `Menu_MGDatabase` → `MainMenuMysteryDB` | Opens Mystery Gift database | `/api/gifts/db` and gift import routes | Candidate. Verify browse/search, gift preview and import. |
| `Menu_EncDatabase` → `Menu_EncDatabase_Click` | Opens encounter database | Phase 13 includes an encounter database; verify the dedicated encounter browser, filtering and encounter-to-editor handoff | Candidate. Verify exact encounter search/filter behavior and encounter-to-editor handoff. |
| `Menu_BatchEditor` → `MainMenuBatchEditor` | Opens batch editor | Box-tools preview/apply and bulk operations exist | Partial. Verify full batch field/action coverage, filters, preview and commit/cancel. |
| `Menu_Folder` → `MainMenuFolder` | Opens folder/file list workflow | Source browse/open and save browser | Partial. Verify recursive browsing, file-type filters and multi-file operations. |

### Options menu and global event wiring

| Control / handler | Desktop behaviour | Web mapping | Assessment |
|---|---|---|---|
| `Menu_Undo` → `ClickUndo` | Undo active save/editor operation | `POST /api/saves/{id}/undo` and history panel | Candidate. Verify operation boundaries and editor refresh. |
| `Menu_Redo` → `ClickRedo` | Redo | `POST /api/saves/{id}/redo` | Candidate. |
| `Menu_Settings` → `MainMenuSettings` | Opens settings editor; reloads program settings | `GET/PUT /api/settings` and settings UI | Candidate. Verify every setting used by Main/SAVEditor/PKMEditor. |
| `Menu_Language` → `AddLanguageMenuItems` / `ChangeMainLanguage` | Runtime language selection and translated controls | `GET /api/languages`, language state and localized data endpoints | Partial. Verify full interface translation, not just localized names/data. |
| `Menu_About` → `MainMenuAbout` | Opens About dialog at shortcuts page | `GET /api/about` and About panel | Partial. Verify version/build information and shortcut/help content. |
| `PB_Legal` → `ClickLegality` | Opens legality report for current editor entity | `GET /api/saves/{id}/pokemon/{slotRef}/legality` and legality panel | Candidate. Verify report details and invalid/fishy classifications. |
| `dragout` → `DragoutDrop`; `Main_DragEnter` / `Main_DragDrop` | Accepts dragged files/entities and exports dragged entity | Browser upload/drop and slot drag/drop | Partial. Desktop OS drag-out and all supported inbound file types need browser-specific design. |
| `C_SAV`/ `PKME_Tabs` event subscriptions | Connect undo/redo, editor preview, legality changes, Showdown import/export, save-file requests | Existing stateful save/editor client | Candidate. Verify event ordering and unsaved-edit protection. |
| Runtime troubleshooting/plugin/developer menu additions | Adds controls after designer initialization | No plugin host equivalent established | Unmapped or explicitly excluded only after plugin compatibility decision. |

## SAVEditor — save workspace

### Primary tabs and embedded editors

| WinForms control | Role | Web counterpart / assessment |
|---|---|---|
| `tabBoxMulti`, `Tab_Box` | Box collection and box navigation | Existing box grid, box selector and save navigation. Candidate. |
| `Tab_PartyBattle` | Party and battle-box workspace | Party and battle-box panels/endpoints. Candidate; verify tab-specific slot counts, ordering and restrictions. |
| `Tab_Other` | Other stored entities/extra slot groups | Existing extra slot and game-extras panels; exact coverage varies by save format. Partial. |
| `Tab_SAV` | Save-specific editor tools | Trainer, inventory, mail, Pokédex, gifts, events and game-extras panels. Partial pending per-button checks. |
| `Box` (`BoxEditor`) | Box slot grid, selection, drag/drop, sorting, context menu, clone and slot mutation | Browser box grid; `GET /api/saves/{id}/boxes`, slot edit/file, swap/move, box layout, bulk and box-tools APIs | Candidate. Verify selection, context commands, clone, sort and undo semantics. |
| `SL_Party` (`PartyEditor`) | Party slots and party-specific stats/ordering | `GET/PUT /api/saves/{id}/party`, party slot edit/file and reorder routes | Candidate. |
| `SL_Extra` (`SlotList`) | Other save-specific stored slot groups | Extra/game-specific panels; no universal mapping for every slot group | Partial; map each format-specific slot group. |

### Daycare

| Control / handler | Desktop behaviour | Web mapping | Assessment |
|---|---|---|---|
| `GB_Daycare` → `SwitchDaycare` | Switches daycare group/view | Daycare panel and `GET /api/saves/{id}/daycare` | Candidate. |
| `dcpkx1`, `dcpkx2` | Display/edit daycare Pokémon slots | Daycare slot file/edit routes | Candidate. |
| `TB_Daycare1XP`, `TB_Daycare2XP` | Display/edit stored experience | Daycare state endpoint | Verify exact editable fields and validation. |
| `DayCare_HasEgg` | Egg available flag | Daycare state endpoint | Candidate; verify update semantics. |
| `TB_RNGSeed` → `UpdateStringSeed` | View/edit daycare RNG seed | No exact seed editor identified in inspected web UI | Unmapped pending capability and format review. |
| `L_XP1/2`, `L_DC1/2`, `L_DaycareSeed` | Labels and context | Browser labels/help | Presentation-only. |

### Save-level tools and controls

The following buttons are functional commands, not decorative controls. Their WinForms handler is the source-of-truth operation; the web feature named in the right column is a candidate, not proof of equivalent behaviour.

| Control / handler | Desktop responsibility | Web counterpart / status |
|---|---|---|
| `B_OpenTrainerInfo` → `B_OpenTrainerInfo_Click` | Trainer identity/profile editor | `/api/saves/{id}/trainer` and Trainer panel — candidate |
| `B_OpenItemPouch` → `B_OpenItemPouch_Click` | Item pouch editor | `/api/saves/{id}/inventory` and pouch routes — candidate |
| `B_OpenBoxLayout` → `B_OpenBoxLayout_Click` | Box layout/background editor | `/api/saves/{id}/boxes/{box}/layout` — candidate |
| `B_OpenWondercards` → `B_OpenWondercards_Click` | Wonder Card/gift editor | `/api/saves/{id}/gifts`, gift file and flag routes — candidate |
| `B_OpenEventFlags` → `B_OpenEventFlags_Click` | Event flag editor | `/api/saves/{id}/events/flags`, preview/apply — candidate |
| `B_OpenPokedex` → `B_OpenPokedex_Click` | Save Pokédex editor | `/api/saves/{id}/pokedex` and species/bulk routes — candidate |
| `B_MailBox` → `B_MailBox_Click` | Mail editor | `/api/saves/{id}/mail` — candidate |
| `B_OpenGiftRibbons` → `B_OpenGiftRibbons_Click` | Gift/ribbon-related save editor | Gift and Pokémon ribbon editor candidates; exact scope needs source-handler trace |
| `B_OpenMiscEditor` → `B_OpenMiscEditor_Click` | Generation-specific miscellaneous save editor | Game-extras feature panels — partial |
| `B_OpenRTCEditor` → `B_OpenRTCEditor_Click` | RTC/clock editor | Gen 3 RTC feature candidate — verify exact controls and save mutation |
| `B_OpenRaids`, `B_Raids`, `B_RaidsDLC1`, `B_RaidsDLC2`, `B_RaidsSevenStar` → `B_OpenRaids_Click` | Raid editor variants | No exact raid editor mapping confirmed in this audit — unmapped pending capability review |
| `B_OpenBattlePass` → `B_OpenBattlePass_Click` | Battle Pass editor | Gen 4 Battle Pass feature candidate |
| `B_Blocks` → `B_Blocks_Click` | Block data editor | Phase 15 includes a block editor; verify the exact save-block editing workflow and format coverage |
| `B_OtherSlots` → `B_OtherSlots_Click` | Additional slot groups | Extra slots/game-extras candidate — partial |
| `B_OpenFashion` → `B_OpenFashion_Click` | Fashion editor | Gen-specific game-extras candidate — verify exact save formats |
| `B_Donuts` → `B_Donuts_Click` | Donut editor | Gen 9 game-extras candidate — verify |
| `B_OpenGlobalLink` → `B_OpenGlobalLink_Click` | Global Link editor | Gen 5 Global Link feature candidate — verify |
| `B_OpenGeonetEditor` → `B_OpenGeonetEditor_Click` | Geonet editor | Gen 4 Geonet feature candidate — verify |
| `B_OpenUnityTowerEditor` → `B_OpenUnityTowerEditor_Click` | Unity Tower editor | Gen 5 Unity Tower feature candidate — verify |
| `B_OpenJoinAvenueEditor` → `B_OpenJoinAvenueEditor_Click` | Join Avenue editor | No dedicated web parity verified — unmapped |
| `B_OpenHoneyTreeEditor` → `B_OpenHoneyTreeEditor_Click` | Honey tree editor | Gen 4 honey-tree feature candidate — verify |
| `B_OpenUGSEditor` → `B_OpenUGSEditor_Click` | Underground editor | Gen 4 Underground feature candidate — verify |
| `B_OpenPokeathlon` → `B_OpenPokeathlon_Click` | Pokéathlon editor | Gen 4 Pokéathlon feature candidate — verify |
| `B_OpenSafariZone` → `B_OpenSafariZone_Click` | Safari Zone editor | Gen 4 Safari Zone feature candidate — verify |
| `B_OpenBerryField` → `B_OpenBerryField_Click` | Berry field editor | Gen 3 berry/field feature candidate — verify |
| `B_OpenPokeblocks` → `B_OpenPokeblocks_Click` | Pokéblock editor | Gen 3 Pokéblock feature candidate — verify |
| `B_OpenSecretBase` → `B_OpenSecretBase_Click` | Secret Base editor | Gen 3 Secret Base feature candidate — verify |
| `B_Roamer` → `B_Roamer_Click` | Roamer editor | Gen 3 roamer feature candidate — verify |
| `B_OpenFriendSafari` → `B_OpenFriendSafari_Click` | Friend Safari editor | Phase 14 has generation-specific editors; exact control-to-feature mapping remains unverified |
| `B_OpenChatterEditor` → `B_OpenChatterEditor_Click` | Chatter editor | No exact web counterpart verified — unmapped |
| `B_OpenMedalsEditor` → `B_OpenMedalsEditor_Click` | Medals editor | Pokémon medal editor exists as nested editor; save medals workflow still needs verification |
| `B_OpenOPowers` → `B_OpenOPowers_Click` | O-Power editor | No exact web counterpart verified — unmapped |
| `B_OpenPokepuffs` → `B_OpenPokepuffs_Click` | Poké Puff editor | No exact web counterpart verified — unmapped |
| `B_OpenSuperTraining` → `B_OpenSuperTraining_Click` | Super Training editor | Pokémon editor's nested Super Training regimen is not necessarily equivalent; save-level workflow unverified |
| `B_OpenPokeBeans` → `B_OpenPokeBeans_Click` | Poké Beans editor | No exact web counterpart verified — unmapped |
| `B_CellsStickers` → `B_CellsStickers_Click` | Stickers/cells editor | No exact web counterpart verified — unmapped |
| `B_OpenApricorn` → `B_OpenApricorn_Click` | Apricorn editor | Gen 4/2 feature candidate — verify |
| `B_OpenSealStickers` → `B_OpenSealStickers_Click` | Seal/sticker editor | No exact web counterpart verified — unmapped |
| `B_Poffins` → `B_Poffins_Click` | Poffin editor | Gen 4 Poffin feature candidate — verify |
| `B_FestivalPlaza` → `B_FestivalPlaza_Click` | Festival Plaza editor | No exact web counterpart verified — unmapped |
| `B_OUTPasserby` → `B_OUTPasserby_Click` | Passerby editor | No exact web counterpart verified — unmapped |
| `B_DLC` → `B_DLC_Click` | DLC editor | Generation-specific DLC feature candidate — verify |
| `B_OpenHallofFame` → `B_HallofFame_Click` | Hall of Fame editor/viewer | Game-extras Hall of Fame candidate; verify read/write vs read-only semantics |
| `B_OpenGear` → `B_OpenGear_Click` | Gear editor | Gen 4 Gear feature candidate — verify |
| `B_OpenLinkInfo` → `B_LinkInfo_Click` | Link information editor | No exact web counterpart verified — unmapped |
| `B_JPEG` → `B_JPEG_Click` | Save box wallpaper/image export | No exact counterpart verified — unmapped |
| `B_MyWallpaper` → `B_MyWallpaper_Click` | Custom wallpaper selection/import | No exact counterpart verified — unmapped |
| `B_SaveBoxBin` → `B_SaveBoxBin_Click` | Export PC box binary | Box export endpoint candidate; verify exact binary format and naming |
| `B_VerifyCHK` → `ClickVerifyCHK` | Verify/fix save checksum | No dedicated checksum verification action identified — unmapped |
| `B_VerifySaveEntities` → `ClickVerifyStoredEntities` | Validate stored entities in save | No equivalent whole-save entity audit identified — unmapped |
| `B_ConvertKorean` → `B_ConvertKorean_Click` | Korean save conversion | No equivalent conversion workflow confirmed — likely gap; inspect whether any format conversion utility covers this exact operation |
| `B_SearchBox` → `B_SearchBox_Click` | Search/filter current box | `/api/saves/{id}/search` and frontend search — candidate |
| `B_PopoutBox`, `Menu_PopoutBoxSingle`, `Menu_PopoutBoxAll` | Pop out one/all box viewers | No separate pop-out browser view verified; likely a desktop-only layout affordance unless a useful multi-window browser equivalent is required |
| `Menu_ExportBAK` → `Menu_ExportBAK_Click` | Export backup | Backup creation/download workflow — candidate |
| `CB_SaveSlot` → `UpdateSaveSlot` | Select save slot where format supports multiple slots | No equivalent selection interaction verified — unmapped pending save capability review |
| `tabBoxMulti` → `ClickBoxSort` | Box tab click/sort interaction | Browser box navigation and sorting; exact sort parity unverified |
| `TB_RNGSeed` → `UpdateStringSeed` | Validates/normalizes daycare seed on commit | No exact equivalent verified — unmapped |

### Dynamic SAVEditor interactions

The implementation creates additional event wiring not fully represented by designer event bindings:

- `InitializeEvents`, `InitializeDragDrop`, `EnableDragDrop`: slot/box/tab drag-enter, drag-over, drop, mouse and reorder behavior.
- `ClickSlot`, `ClickBoxDouble`, `ClickClone`, `ClickBoxSort`, `FinishBoxManipulation`: selection, clone, sorting and commit flow.
- `ClickUndo`, `ClickRedo`, `UpdateUndoRedo`: history state and command enablement.
- `ToggleViewBox`, `ToggleViewParty`, `ToggleViewDaycare`, `ToggleViewSubEditors`, `ToggleViewMisc`: save-format/capability-dependent visibility.
- `B_SearchBox_Click`, `CreateSearcher`, `SeekNext`, `SeekPrevious`, `UpdateSearch`, `ApplyNewFilter`: search state, filters and navigation.
- `GetBulkImportSettings`, `GetImportSettingsOverride`, `LoadBoxes`, `ImportGroup`, `ExportSaveFile`, `ExportBackup`: import/export configuration and save/backup behavior.

The web app has box drag/drop, bulk operations, box tools, party reorder, daycare and save capability handling. Each of the above desktop semantics must be compared individually; broad feature similarity is insufficient.

## PKMEditor — Pokémon editor

### Tabs and field groups

| WinForms tab/control group | Fields and controls covered | Web counterpart / assessment |
|---|---|---|
| `Tab_Main` / `Hidden_Main` | Species `CB_Species`; PID `TB_PID`, `BTN_RerollPID`; shiny star/square and `BTN_Shinytize`; nickname `TB_Nickname`, nicknamed flags/warnings; EXP/level and `ExperienceBar`; nature; held item; friendship; form/`FA_Form`; ability and manual ability number; language; egg; PKRS; country/subregion/3DS region; shadow Pokémon/purification/shadow ID; gender | Schema main tab and helper actions (shiny, PID/EC, gender, level/EXP, species/form, ability, item, egg, PKRS, region). Candidate. Verify all format-specific fields, control interdependencies and keyboard shortcuts. |
| `Tab_Met` / `Hidden_Met` | Origin game, battle version, met location, ball, met date/level, fateful encounter, obedience level, ground tile, time of day, egg state/location/date | Schema metadata/met tab. Candidate. Verify legality-driven location lists, date/time constraints, egg/met coupling and origin-format behavior. |
| `Tab_Stats` / `Hidden_Stats` | `Stats` / `StatEditor`, IV/EV/stat alignment, party stats, contest stats, status and format-specific stat fields | Schema stats tab and stat helper actions. Candidate. Verify stat calculation, Hyper Training/AV/awakening or generation-specific semantics and invalid value handling. |
| `Tab_Moves` / `Hidden_Moves` | Four current `MoveChoice` controls; PP and PP Ups; four relearn moves; per-move legality warning indicators; relearn flags; move shop; alpha-mastered move state | Schema moves tab and fix/suggest moves/relearn actions. Candidate. Verify legal move source, move validation, PP healing/PP-Up behavior, relearn flags and Alpha move constraints. |
| `Tab_OTMisc` / `Hidden_OTMisc` | OT/HT names and gender; TID/SID and handler; encryption constant and reroll; HOME tracker; ribbons, medals, history; markings; favorite; origin/battle-version markings; extra bytes; catch rate; obedience; received date/time; records | Schema trainer/cosmetic/advanced fields and nested helper actions. Partial. Verify dedicated ribbons/medals/history/records dialogs and all generation-specific miscellaneous fields. |
| `Tab_Cosmetic` / `Hidden_Cosmetic` | Form argument, size scalars, markings, cosmetic buttons, gender/form-dependent visual fields, status and sprite/ball/origin previews | Schema cosmetic tab and preview. Partial. Verify size/scale, markings, form argument and sprite preview interactions. |

### Core editing and validation semantics

These are functional requirements even when the visible control is a textbox, label or icon:

1. `PreparePKM` validates child controls before producing a clone. `EditsComplete` blocks incomplete entities, locates the invalid tab and supports the deliberate Ctrl+Shift+Alt override. Web save/patch/export actions must not silently bypass equivalent required-field validation.
2. Validation is conditional on entity format/type: species and moves are required; held item starts at format 2; ability/nature/met location/ball at format 3; egg location at format 4; country/subregion for PK6/PK7; relearn moves at format 6; stat alignment at format 8; Alpha mastered state for PA8. Web schema capability and validation must match the PKHeX source, not a single global field list.
3. Combo boxes have distinct `SelectedIndexChanged`, `Leave`, `Validating` and paint/dropdown behavior. A simple PATCH schema type check is not sufficient evidence of parity.
4. Species, form, origin game, language, country, nature, gender, ability, held item, ball and met/egg locations can repopulate or constrain other fields. Verify dependent-field recalculation and legality refresh after each change.
5. Current moves use `LegalMoveSource`; move dropdowns validate against the current encounter/legality context. Relearn moves have separate validation and warnings.
6. EXP and level are synchronized; changing one updates the other. Friendship/level/date/ID fields use validated numeric input and format-specific limits.
7. PKRS infected/cured/strain/days are coupled. Egg and met-as-egg state are coupled. Shadow state, purification and shadow ID are coupled.
8. PID/EC reroll, shiny mode, gender, markings, favorite, origin and battle-version marks, handler, and manual ability selection are actions, not plain fields.
9. Drag/drop is enabled on editor subcontrols at runtime. Verify browser file import and drag/drop affordances separately.
10. Save-file access is requested through `SaveFileRequested`; secret values may be hidden according to the save generation; `HaX` changes validation/visibility. Web capabilities and privacy rules must preserve these distinctions.

### PKMEditor event-to-behaviour map

| Event/handler family | Desktop behaviour | Web counterpart / assessment |
|---|---|---|
| `UpdateEXPLevel`, `ExperienceBar.ValueChanged` | Keep EXP and displayed level synchronized | Schema EXP/level fields; verify bidirectional update |
| `UpdateRandomPID`, `UpdateRandomEC`, `UpdateShinyPID`, `Update_ID`, `Update_ID64` | PID/EC/shiny/ID editing | Helper actions; verify deterministic field updates and preservation rules |
| `UpdateFormArgument`, `UpdateForm`, `RefreshFormArguments`, `UpdateSpecies`, `UpdateOriginGame` | Species/form/origin-dependent fields and legal values | Schema options and PATCH; verify dependency refresh |
| `UpdatePKRSstrain`, `UpdatePKRSdays`, `UpdatePKRSCured`, `UpdatePKRSInfected` | PKRS state coupling | Schema PKRS fields; verify constraints |
| `UpdateIsEgg`, `UpdateMetAsEgg`, `UpdateNickname`, `UpdateIsNicknamed`, `UpdateNotOT` | Egg, nickname and handler state | Schema fields; verify dependent state |
| `ValidateComboBox`, `ValidateComboBox2`, `ValidateMove`, `ValidateLocation`, `ValidateMoveDropDown` | Field/move/location validation and visual warnings | API validation and schema options; parity not yet proven |
| `ClickFriendship`, `ClickLevel`, `ClickGender`, `ClickPP`, `ClickPPUps`, `ClickMarking`, `ClickFavorite`, `ClickOT`, `ClickCT`, `ClickBall`, `ClickMetLocation`, `ClickGT`, `ClickNature`, `ClickMoves` | Context actions and convenience editing | Helper actions and field controls; map each individually |
| `OpenRibbons`, `OpenSuperTrainRegimen`, `OpenHistory`, `B_Records_Click`, `B_MoveShop_Click`, `B_PlusRecord_Click` | Nested ribbon, regimen, history, records and move-shop workflows | Some editor helper actions exist; dedicated web counterparts and cancel/save semantics remain unverified |
| `ChangeHandlerIndex`, `ChangeSelectedTabIndex`, `ToggleSecrets`, `SetPKMFormatMode`, `FinalizeInterface` | Format/handler/tab/secret-value dependent UI | Schema capabilities and tab rendering; verify all format gates |
| `UpdateLegality`, `UpdateSprite`, `UpdateStats`, `SetSuggestedMoves`, `SetSuggestedRelearnMoves`, `SetSuggestedMetLocation` | Legality, sprite, stat and suggestion refresh | Legality endpoint, preview and helper actions; verify refresh timing and stale-state handling |
| `EnableDragDrop`, `LoadShowdownSetDefault` | Drag/drop and Showdown import integration | Browser import and Showdown feature; exact behavior unverified |

## Known web route/component inventory relevant to these controls

Inspected web-host routes and frontend functions include:

- Save lifecycle: `/api/saves`, `/api/saves/{id}/download`, `/api/saves/{id}/original`, `/api/source/browse`, `/api/source/open`.
- Save workspace: `/api/saves/{id}/boxes`, `/party`, `/battlebox`, `/daycare`, `/daycare/state`, `/slots/move`, box layout/import/export and box-tools preview/apply.
- Pokémon: slot edit/file routes, `/pokemon/{slotRef}/schema`, PATCH `/pokemon/{slotRef}`, helper `/pokemon/{slotRef}/actions/{name}`, legality routes.
- Save-specific: `/trainer`, `/inventory`, `/mail`, `/records`, `/gifts`, `/events/flags`, `/events/preview`, `/events/apply`, `/pokedex`.
- History/configuration: `/history`, `/undo`, `/redo`, `/settings`, `/about`, `/languages`.
- Frontend functions include `renderSchema`, `showEditor`, `loadBoxes`, `loadParty`, `loadBattleBox`, `loadDaycare`, `loadPlayerData`, `renderTrainer`, `renderInventory`, `renderMail`, `renderGifts`, `renderEvents`, `renderRecords`, `loadPokedex`, `showTransferWizard`, `renderHistory`, `loadSaveCapabilities` and game-extras panels.

These routes/functions establish candidate destinations only. They do not establish that every desktop command, field, validation rule or edge case is implemented.

## Gap register from this first detailed pass

### High-priority candidates to verify first

1. Main standalone PKM export and Showdown import/export for current Pokémon, party and current box.
2. Main report grid, encounter database, QR workflow, drag-out and plugin/troubleshooting commands.
3. SAVEditor save checksum verification and stored-entity verification.
4. SAVEditor save-slot selector, daycare RNG seed, box pop-out, wallpaper/JPEG, Korean conversion and exact box binary export.
5. SAVEditor game-specific editors with no confirmed web route/component: raids, blocks, Friend Safari, Chatter, O-Powers, Poké Puffs, Poké Beans, stickers/cells, Festival Plaza, passerby, Link Info, Join Avenue and any others not represented by feature capability metadata.
6. PKMEditor full validation parity, format-dependent visibility, move legality source, met/egg dependency handling, PKRS coupling, and nested ribbons/medals/history/records/move-shop/regimen flows.

### Evidence still required before status can be upgraded

- For each candidate, inspect the exact frontend control and the endpoint implementation, not only route existence.
- Trace the WinForms handler body to its Core operation and compare it with the web handler's Core operation.
- Test capability gating across representative save generations and formats.
- Test valid, invalid, boundary and incomplete input; cancellation/no-op; stale state; undo/redo; and persisted save mutation.
- Verify exported files can be reopened by PKHeX and that original save bytes remain unchanged until an explicit commit/export.
- Update `control-audit.csv` and `window-mapping.csv` with evidence and status only after these checks.

## Current conclusion

Main, SAVEditor and PKMEditor all have broad web counterparts, but the control-by-control comparison identifies several probable gaps and many unverified workflows. No surface is marked parity-complete. The next implementation task should be selected only after the high-priority candidates above are checked against the exact web UI and handler bodies.
