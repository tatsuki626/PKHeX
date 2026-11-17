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
| `UpdateSaveSlot` — SAV4BR slot switch, closes battle-pass editor, resets box names and reloads viewers | `GET/PATCH /saves/{id}/slot-selection`; browser four-slot selector refreshes active save panels | **Partial** — slot selection, range validation and view refresh are implemented; BattlePass editor parity and explicit dependent-view invalidation remain open |
| `UpdateStringSeed` / `SetDaycareSeed` — strict hexadecimal validation; supports 16/32/64/128-bit daycare RNG seeds; marks save edited | `GET /saves/{id}/daycare` returns typed seed per group; `PATCH /saves/{id}/daycare/seed` writes through the matching PKHeX interface; browser seed field | **Implemented; local CI passed (2026-11-17)** — exact-width hexadecimal validation, per-group capability gating, audit and save persistence; tests cover Gen4/5/7 and Emerald Gen3 32-bit, including the Gen5 BW-vs-B2W2 capability distinction |

## Daycare

Web has `GET /saves/{id}/daycare`, per-slot file upload/delete, `PATCH /saves/{id}/daycare/state` for experience/egg availability, and `PATCH /saves/{id}/daycare/seed` for typed RNG seed writes. The GET response includes a per-group {bits, hex} seed only when the group implements one of PKHeX's `IDaycareRandomState<T>` interfaces. The browser renders an exact-width hexadecimal field and reloads after a successful write. Experience is exposed per group only when that group's `IDaycareStorage` also implements `IDaycareExperience`.

Desktop `SAVEditor.SwitchDaycare` cycles to the next group, asks for Yes/No confirmation, and only changes `DaycareIndex` after acceptance. It then calls `ResetDaycare`, refreshing the current group's occupancy, Pokémon, experience and seed controls. `GetCurrentDaycare` resolves `IDaycareMulti[index]` (or the single `IDaycareStorage`).

The host now mirrors this behaviour:
- stores the selected daycare group per open save and resets it to group 0 when the user opens a different save;
- exposes a cyclic switch control only for saves with multiple groups;
- requires confirmation before changing the selected group; cancel preserves the current group;
- renders only the selected group's slots, experience and RNG seed, instead of showing all groups simultaneously;
- retains group-indexed upload/delete/seed APIs; switching is view state and does not create a save mutation or undo entry.

The ORAS `SAV6AO` fixture is the concrete two-group case (`DaycareCount == 2`). Smoke coverage checks groups 0 and 1, invalid group rejection, and the browser confirmation/selected-group contract.

**Result: implemented on host; local CI pending.**

## Generation-specific save tools

Desktop handlers dispatch into many independent forms, often by concrete save type:
- Trainer, inventory, Pokédex, Mystery Gifts, event flags/work, records and mail.
- Box layout, link info, berries, secret bases, Pokéblocks, Poké Puffs, Super Training, Hall of Fame, passerby, DLC, Poké Beans, stickers, honey trees, Friend Safari, RTC, Geonet, Unity Tower, Join Avenue, Safari Zone, Pokéathlon, medals, chatter, roamer, Festival Plaza, apricorns, gift ribbons, raids, blocks, battle pass, gear, fashion, donuts and global link.
- Search/pop-out tools and conversion helpers (including Korean conversion).

The browser has confirmed general routes/UI for trainer, inventory, Pokédex, gifts, event flags/work, records, mail, save blocks, raids and fashion-related areas. This is **not** proof that every desktop editor's fields, generation gates, modal commit/cancel behaviour or validation is represented. Tools without a clearly matching browser operation remain **unmapped pending per-window trace**, not automatically excluded.

## Required implementation/verification order

1. Checksum diagnostics and stored-entity bulk verification are implemented and passed the complete local CI suite on 2026-11-17. The CI suite covers valid/invalid/edited checksum response paths and stored-entity BulkAnalysis response shape; retain representative real-save report comparison as follow-up parity evidence.
2. SAV4BR four-slot selection and typed daycare RNG seed editing passed the complete local CI suite on 2026-11-17. Remaining SAV4BR work is BattlePass editor parity and explicit dependent-view invalidation. Multi-group daycare switching/confirmation is implemented on the host and awaits local CI.
3. Compare box import/export and save export pre-write cleanup/validation.
4. Map each generation-specific sub-editor to its existing web panel/API; record field-level omissions and modal semantics.
5. Test cancel vs commit, undo/redo, save reload and checksum state for every mutating workflow.

Status rule: a route is only a candidate until its UI, validation, capability gating, persistence and read-after-write behaviour are verified.
