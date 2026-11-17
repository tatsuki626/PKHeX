# SAVEditor — handler-to-web verification

Reference baseline: PKHeX source commit `9a1a8f4a450996e9572ee4428e9ae709166c2c6d`.
Host: `experiment/upstream-winforms-web-port`; inspected `Program.cs` and `wwwroot/js/app.js`.
This is static source tracing, not runtime parity certification.

## Core storage and slot workflows

| Desktop responsibility / handler | Web path found | Result |
|---|---|---|
| Box/party/battle-box/daycare/misc slot rendering and updates | `GET /saves/{id}/boxes`, `/party`, `/battlebox`, `/daycare`; slot PATCH/file routes | **Partial** — corresponding web surfaces exist; validate every slot class, read-only rules, empty/egg semantics and format-specific extra slots |
| Drag/drop, box sorting, tab movement, pop-out box/list/viewer, search | Box grid, bulk move, search and box layout UI | **Partial** — desktop includes tab drag/drop sorting and detached views; no complete interaction equivalence proven |
| `ClickUndo` / `ClickRedo` | `POST /saves/{id}/undo`, `/redo` | **Candidate** — verify slot changelog scope and undo grouping against desktop |
| `LoadBoxes` | `POST /saves/{id}/boxes/{box}/import`, `GET /boxes` | **Partial** — desktop supports folder import, clear-all, overwrite policy and format conversion; browser ZIP/import flow must match these options |
| `B_SaveBoxBin_Click` | Box export endpoints | **Partial** — desktop exports current box binary; browser exports are file/format-oriented, exact binary parity unproven |
| `ExportSaveFile` | `GET /saves/{id}/download` plus export dialog | **Partial** — desktop validates child controls and may run `IStorageCleanup.FixStoragePreWrite()` before writing; host download path must preserve pre-write cleanup and force-save-as options |
| `ExportBackup` | Backup download/restore APIs | **Partial** — desktop copies the original file from source path; web backup workflow semantics differ and need comparison |

## Save integrity and verification

| Desktop operation | Web evidence | Result |
|---|---|---|
| `ClickVerifyCHK` — refuse checksum verification if save is edited; report valid checksums; optionally copy checksum details for invalid save | `GET /saves/{id}/verification/checksum`; browser Verify checksums control; durable `SaveEntry.HasBeenEdited` marker survives restart | **Implemented; local CI passed (2026-11-17)** — valid/invalid/edited states and invalid checksum details are exposed without mutating the save |
| `ClickVerifyStoredEntities` — `Core.Bulk.BulkAnalysis` over all stored entities; report clean or offer detailed localized report copy | `GET /saves/{id}/verification/stored-entities` calls `PKHeX.Core.Bulk.BulkAnalysis`; browser Verify stored Pokémon control renders localized report and copy action | **Implemented; local CI passed (2026-11-17)** — includes box/party/other storage discovered by PKHeX.Core; separate from box-only legality report |
| `UpdateSaveSlot` — SAV4BR slot switch, closes battle-pass editor, resets box names and reloads viewers | No battle-pass/current-save-slot selector or matching API route found | **Missing** — generation-specific multi-slot selection and dependent editor invalidation |
| `UpdateStringSeed` / `SetDaycareSeed` — strict hexadecimal validation; supports 16/32/64/128-bit daycare RNG seeds; marks save edited | Daycare endpoint supports egg state and experience; no RNG seed field or write route found | **Missing** — expose capability-typed seed value and exact-width hex validation |

## Daycare

Web currently has `GET /saves/{id}/daycare`, per-slot file upload/delete, and `PATCH /saves/{id}/daycare/state` for experience and egg availability. The frontend renders daycare groups and applies egg-state changes.

Desktop additionally:
- supports one or multiple daycare groups and explicit group switching with confirmation;
- displays occupancy and per-slot experience where supported;
- exposes generation-specific random seed values (UInt16/UInt32/UInt64/UInt128);
- exposes egg availability only for `IDaycareEggState`;
- uses save-native daycare interfaces, not a generic Pokémon field.

**Result: partial.** The browser currently has basic slot/egg/experience workflows, but group switching and RNG seed parity are absent from the inspected UI/API.

## Generation-specific save tools

Desktop handlers dispatch into many independent forms, often by concrete save type:
- Trainer, inventory, Pokédex, Mystery Gifts, event flags/work, records and mail.
- Box layout, link info, berries, secret bases, Pokéblocks, Poké Puffs, Super Training, Hall of Fame, passerby, DLC, Poké Beans, stickers, honey trees, Friend Safari, RTC, Geonet, Unity Tower, Join Avenue, Safari Zone, Pokéathlon, medals, chatter, roamer, Festival Plaza, apricorns, gift ribbons, raids, blocks, battle pass, gear, fashion, donuts and global link.
- Search/pop-out tools and conversion helpers (including Korean conversion).

The browser has confirmed general routes/UI for trainer, inventory, Pokédex, gifts, event flags/work, records, mail, save blocks, raids and fashion-related areas. This is **not** proof that every desktop editor's fields, generation gates, modal commit/cancel behaviour or validation is represented. Tools without a clearly matching browser operation remain **unmapped pending per-window trace**, not automatically excluded.

## Required implementation/verification order

1. Checksum diagnostics and stored-entity bulk verification are implemented and passed the complete local CI suite on 2026-11-17. The CI suite covers valid/invalid/edited checksum response paths and stored-entity BulkAnalysis response shape; retain representative real-save report comparison as follow-up parity evidence.
2. Add multi-slot SAV4BR selection and daycare RNG seed editing, guarded by save capabilities.
3. Compare box import/export and save export pre-write cleanup/validation.
4. Map each generation-specific sub-editor to its existing web panel/API; record field-level omissions and modal semantics.
5. Test cancel vs commit, undo/redo, save reload and checksum state for every mutating workflow.

Status rule: a route is only a candidate until its UI, validation, capability gating, persistence and read-after-write behaviour are verified.
