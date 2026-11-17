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
