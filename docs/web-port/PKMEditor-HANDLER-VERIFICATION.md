# PKMEditor — field and handler-to-web verification

Reference baseline: PKHeX source commit `9a1a8f4a450996e9572ee4428e9ae709166c2c6d`.
Host: `experiment/upstream-winforms-web-port`; inspected `Program.cs` and `wwwroot/js/app.js`.
This pass traces the WinForms editor's core event families against the schema-driven editor. It is not a claim of full field parity.

## Existing web architecture

The host exposes:
- `GET /saves/{id}/pokemon/{slotRef}/schema?advanced=true` — schema built from the actual PKM and format/context.
- `PATCH /saves/{id}/pokemon/{slotRef}` — apply editable field patch to a cloned save and persist.
- `POST /saves/{id}/pokemon/{slotRef}/actions/{name}` — PKHeX-backed helper action.
- `GET /saves/{id}/pokemon/{slotRef}/legality` — legality analysis.
- Slot references cover box, party, battle box and daycare.

The frontend `renderSchema()` builds tabs and controls from returned schema metadata, filters unavailable controls, and submits patches. `actionAvailable()` gates helper actions using schema-supported actions and interfaces. This is a sound reuse-first foundation, but the schema must be audited field-by-field against `GetFieldsfromPKM` / `GetPKMfromFields` and the WinForms controls.

## Handler families and parity findings

| WinForms handlers / field family | Desktop semantics | Web counterpart | Result |
|---|---|---|---|
| `PreparePKM`, `GetInvalidParentTab`, `IsNotValid` | Force validation of controls, reject incomplete/invalid fields and return a clone | Schema validation + PATCH error responses | **Partial** — no evidence yet of equivalent cross-field validation/focus-to-invalid-tab behaviour |
| `SetPKMFormatMode`, `LoadFieldsFromPKM` | Switch typed getter/setter bindings and control context by PKM format; optionally convert compatibility | Schema is generated per PKM format/context and exposes origin generation/format | **Candidate** — verify all formats and conversion/refusal paths |
| Species/form/ability/origin game | Species changes recalculate EXP, ability list, forms, gender and nickname; origin changes met-location legality and Gen4 ground-tile visibility | Schema options and PATCH; legality endpoint | **Partial** — verify dependent option refresh and no stale IDs after species/version changes |
| EXP/level | Editing either recalculates the other using species growth curve and clamps max level (except HaX path) | EXP and level fields in schema; browser fields | **Partial** — exact bidirectional clamping and growth-curve parity not proven |
| Moves/relearn moves | Validates each selection, heals PP, updates PKM move/relearn arrays and legality; suggestions require confirmation; random suggestions use modifier key | Schema move fields and move-suggestion/helper actions | **Partial** — verify PP preservation, alpha/mastery moves, invalid-line feedback and confirmation semantics |
| PID/EC/shiny/gender/nature | Rerolls PID/EC; PID may be constrained by gender/nature/ability; shiny mode may update PID; older formats have different rules | Helper actions for reroll PID/EC, shiny modes and gender fix | **Partial** — verify exact format-specific constraints and resulting fields |
| Met/egg locations, dates, ball, met level, fateful encounter | Validates contextual location lists and updates legality; egg and met fields have dependent visibility | Schema fields/options and legality recalculation | **Partial** — verify game-context filtering and all date/location invariants |
| Pokerus, shadow/purification, country/region, handler/OT | Typed interface-specific state and dependent controls | Schema fields/actions for supported interfaces | **Partial** — interface support is not proof of all field semantics |
| Ribbons/medals/history/favorite/markings/affixed ribbon | Opens nested editors or mutates specialized ribbon/marking interfaces | Ribbon and marking helper actions; some metadata schema fields | **Partial** — nested editors and full ribbon/medal collection parity unverified |
| Extra bytes, HOME tracker, battle version, move flags, obedience, form arguments, alpha move mastery | Format-specific advanced metadata and special interfaces | Advanced schema and interface-gated actions | **Partial** — audit each concrete format; read-only advanced display is not editable parity |
| Tab changes and warning controls | Tracks invalid tab, nickname/OT warnings and font limitations | Schema tabs and basic browser validation | **Partial** — warning parity and navigation to invalid field unverified |

## Confirmed implementation gaps to validate

1. **Validation contract:** PATCH must reject invalid cross-field combinations with field-addressable errors; client should focus the affected field/tab. Do not rely only on HTML input constraints.
2. **Dependent field refresh:** species, origin game, form, ability, met/egg location, ball and level/EXP changes must refresh dependent options and legality without stale values.
3. **Move semantics:** verify current/relearn moves, PP, PP Ups, alpha move/mastery, technical records and move-shop flags as separate state.
4. **Format matrix:** test representative PK1–PK9 records and origin-generation combinations; format is not the same as origin generation.
5. **Nested editors:** ribbon/medal/history/move-shop and other interface-backed dialogs must either have explicit browser counterparts or be marked excluded with rationale.
6. **Advanced fields:** verify which fields are editable vs intentionally read-only; do not count read-only visibility as full parity.

## Targeted tests required

- GET schema for each PK1–PK9 format and multiple origin generations; assert expected tabs, field types, option sets and supported actions.
- PATCH one field per tab; verify read-after-write and that unrelated bytes/fields remain unchanged.
- Invalid values, cross-field conflicts and unsupported fields must be rejected without mutation.
- Species/form/origin changes must refresh dependent options and preserve only legal compatible selections.
- EXP↔level round-trip across growth curves and max-level boundary.
- Move and relearn-move changes must preserve PP rules and update legality.
- PID/EC/shiny actions must obey generation-specific constraints.
- Nested editor changes must persist only on explicit apply and cancel without mutation.

Status rule: schema presence or an action name alone is not parity. Require a field mapping, format-specific behaviour and read-after-write test.


## Validation implementation update (2026-11-17; CI pending)

The host has begun closing validation contract item 1:
- PATCH validation errors include `field` and `tab` metadata, preserved by the browser API client and displayed in the editor error message.
- Explicit invalid forms and attempts to clear species in an occupied slot are rejected rather than silently normalized.
- Species is applied before Form/Ability so dependent validation is independent of JSON property order. If a species change invalidates the previous form and no new Form was supplied, the form resets to zero.
- Simultaneous Level and EXP changes must agree; Level-only changes continue to calculate EXP with the species growth curve.
- Smoke tests cover these rejection paths and assert rejected requests leave the slot unchanged.

This does not yet provide WinForms-equivalent focus-to-control/tab navigation, nor does it close the remaining dependent-option and generation-specific validation gaps. The full local CI suite passed at `2026-11-17T10:45:41+04:00` on host commit `a0f2fb2942558cb1464ba44fd9dc147b2cc001af`.

### Level/EXP order-independent validation refinement (2026-11-18; local CI verified)

The schema PATCH validator now checks a supplied Level/EXP pair against the species growth curve before applying either value. This prevents JSON property order from silently deciding which value wins. Level bounds use PKHeX's `Experience.MinLevel` and `Experience.MaxLevel`; editing Level or EXP caps excess maximum-level EXP to the growth-curve threshold, matching the desktop editor's normal non-HaX behavior (the web editor has no HaX override). Smoke tests cover conflicting values in both JSON property orders and the maximum-level cap. Host commits: Program.cs `2fcd855` with final-species growth-curve refinement `8b08737`, smoke tests `c24508e`. The pair check runs after Species is applied, and retains the explicitly supplied EXP once validated. This is a targeted correctness closure; invalid combo navigation, move learnability validation and broader per-format field constraints remain open.


## Move PP validation implementation update (2026-11-17; local CI verified at 10:45 +04:00)

The host schema PATCH validator now:
- Rejects PP or PP Ups on an empty move slot.
- Rejects current PP above the move's context-specific base PP adjusted by PP Ups, using PKHeX.Core `MoveInfo.GetPP`.
- Returns the offending PP/PP Ups field and identifies the Moves tab.
- Includes smoke coverage for empty-slot PP and excessive PP, alongside the existing no-mutation-on-rejection check.

Host commits: `0285d33`, `eacffa0` and `189e9e8` (the latter adds explicit before/after move/PP/PP Ups immutability checks). This is a storage-integrity guard, not a full reproduction of WinForms `ValidateMove` / `LegalMoveSource`: legality coloring, deferred legal move lists, heal-on-selection, relearn move semantics and alpha/mastery handling remain open. Local CI is pending.


## PID / Encryption Constant editing update (2026-11-17; local CI verified at 13:07 +04:00)

An opt-in Identity schema tab (`advanced=true`) now exposes raw PID editing for format 3+ and Encryption Constant editing for format 6+, both as unsigned 32-bit values. Unsupported fields are omitted by format. The error-tab map routes validation failures to Identity.

This is distinct from helper actions: reroll PID, reroll EC, PID-by-nature and shiny generation continue to call PKHeX.Core methods that enforce their own correlation rules. Raw field editing writes only the selected field. Smoke tests cover PID/EC round trips and generation gating. Complete local CI passed at `2026-11-17T13:07:26+04:00` on tested host commit `db7cf271226029a7c47326ee0ca1d123d9a1a654`. A stale invalid-PID test value was updated to exercise UInt32 overflow rather than a now-valid editable PID. Report: `reports/local-ci/pkhex-local-ci-20261117-130437-200621630.md`.

## Dependent choices and origin/location validation update (2026-11-17; host implementation, local CI pending)

The host adds `GET /saves/{id}/pokemon/{slotRef}/dependencies`, deriving:
- Form labels from PKHeX `FormConverter` for the selected species and context.
- Ability IDs/names from the selected form's `IPersonalAbility` data, including the Gen5 Basculin form-specific Reckless option.
- Origin-version options limited to supported versions no newer than the PKM format.
- Met and egg location options from PKHeX's `GameInfo.GetLocationList` for the resolved origin version and PKM context.

The browser refreshes the ability list when Species changes, and refreshes form/location options when Form or Version changes. Schema PATCH now accepts earlier origin generations for transferred Pokémon, rejects unsupported/future origin versions, validates explicit met/egg locations against the contextual PKHeX lists, and normalizes stale stored locations after an origin-only change. Format-specific schema PATCH errors navigate to the reported tab and focus the invalid field. The legacy basic `/edit` route remains unchanged to preserve existing move-healing semantics.

Smoke tests cover the dependency response, invalid species and future origin versions, invalid met locations, and no mutation after rejection. Local CI is pending.
Host implementation commits: Program.cs `2147c79`, `8c0d194`, `4d2708b`; browser `a40de15`, `a9307b7`, `5023e96`, `ae980a6`, `03313b0`, `e6a5598`, `2295c15`; smoke tests `6f7d010`, `e9b3258`. Full local CI remains pending.
CI follow-up: the first local run failed at compile time because the ability-name collection is `IReadOnlyList<string>`; host fix `74e9204` changes `.Length` to `.Count`. Report `pkhex-local-ci-20261117-140610-513710217.md` was confirmed on origin. Rerun local CI; no tests completed in the failed run.


Legacy-format follow-up (2026-11-17): the published 15:05 host CI report reached smoke test 1340, where the Generation 1 helper fixture failed while round-tripping `Version`. PK1/PK2 do not store origin-game metadata; their inherited property is `GameVersion.Invalid`, not a selectable saved game. Host fix `04983db` keeps `Version` visible but read-only for PKM formats below 3, and smoke commit `c71a430` asserts that capability. This preserves strict origin validation for formats that actually store a version. Full local CI is still pending; do not mark G-013/G-015 complete until a PASS report is published.

Implementation refinement: host commit `3db7760` applies the format-3 origin-version gate after interface-property discovery, ensuring PK1/PK2 `Version` remains read-only even if an interface exposes a setter.


CI verification: complete local CI passed at 2026-11-18T05:44:29+04:00 on host commit `647c4139972634aaf73f1ff335c5baaeda382e63`. Report: `reports/local-ci/pkhex-local-ci-20261118-054137-881046641.md`. This verifies the Level/EXP property-order checks and level-100 EXP normalization. G-012 remains open for incomplete ComboBox validation/focus parity and broader format-specific constraints.


### G-012 — PKMEditor ComboBox selection parity (implementation partial; local CI pending)

Host commits `f05d171` and `c7d2c0f` add server-side option-name validation for Species, Held Item, Nature, Stat Alignment, regular/relearn moves, and Alpha Move. Errors retain field/tab metadata; Stat Alignment maps to Stats and Alpha Move maps to Moves. The browser already switches to the reported tab, focuses the matching field, sets `aria-invalid`, and announces the error. The validator deliberately does not reject a move merely because PKHeX legality says it is unlearnable. Smoke commit `2ca0827` covers invalid move selection and non-mutation of move/PP/PP Ups. Local CI pending.

Still open: Ball and PK6/PK7 Country/SubRegion option-list validation, complete format/interface gate coverage, and byte-level atomicity checks for ordinary slots and staged Battle Pass records. G-012 remains open.


CI follow-up (2026-11-18): the first run reached the new combo validator but a valid Nature PATCH threw because `Enum.IsDefined(Type, object)` received `Int32` for byte-backed `Nature`. Host commit `efe8424` now range-checks and casts to `byte`; smoke commit `f7497f9` adds a valid Nature PATCH regression. The failed run is published at `reports/local-ci/pkhex-local-ci-20261118-061604-772179726.md`. Re-run host local CI; do not treat this slice as verified yet.

CI follow-up (2026-11-18): the byte-backed Nature enum exception is fixed in host commit `efe8424`. The next CI run reached a successful PATCH but failed because the added smoke test expected an `ok` property; the actual PATCH response contract is `{ pokemon, fieldsUpdated }`. Host smoke commit `8a5f30a` now asserts the returned `pokemon.natureId`. Failed report: `reports/local-ci/pkhex-local-ci-20261118-061957-877282615.md`. Local CI rerun pending.

CI verification update (2026-11-18): the partial ComboBox-validation slice passed complete host local CI at `2026-11-18T06:51:40+04:00`, tested host commit `e5d4377a0049a5a1f28591a98761f33aef8afd9f`. Report: `reports/local-ci/pkhex-local-ci-20261118-064844-908788328.md`. The valid byte-backed Nature PATCH and existing invalid move/non-mutation smoke checks now pass. This verifies only the implemented subset; G-012 remains open for Ball and PK6/PK7 Country/SubRegion option validation, full format/interface gates, and stronger serialized-slot/staged Battle Pass atomicity checks.


### G-012 follow-up — Ball and 3DS region validation implemented (2026-11-18; local CI pending)

Host commits `688b06e`, `f808916` and `d9f4a24` extend the shared schema PATCH validator to check Ball selections against PKHeX's `GameDataSource.BallDataSource` option IDs, bounded by the active PKM's `MaxBallID` and, for the desktop PK6/PK7 `IRegionOrigin` fields, validate Country IDs with `GeoLocation` and Region IDs against the selected country's region table. The `Region` property is the Core/schema name for the desktop SubRegion ComboBox. Both Country and Region errors navigate to the Trainer tab. Smoke commit `1cad792` adds invalid Country/Region rejection on the Gen 6 PKM fixture and checks both fields remain unchanged after rejection. The Ball and Country/Region implementation has not yet passed local CI; do not mark G-012 complete. Full generation/interface gate coverage and serialized-slot/staged Battle Pass byte-level atomicity remain outstanding.


CI follow-up (2026-11-18): report `pkhex-local-ci-20261118-070302-355277705.md` failed at the schema PATCH smoke on a legacy Gen 3 fixture whose Ball value is `0`; the initial option-list check treated that stored sentinel as invalid even when the edit did not change Ball. Host fix `7f6ee59` preserves zero as the legacy/no-ball sentinel while still validating every nonzero Ball against PKHeX's option IDs and the active format limit. This correction is not yet CI-verified; rerun local CI. G-012 remains open.
