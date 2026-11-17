# WinForms → ASP.NET Web Port

## Project decision

This is a **reuse-first compatibility port**: preserve and reuse existing PKHeX code wherever technically possible, and convert only the Windows-specific interface or code that cannot run in the web host. It is not a rewrite.

Every user-facing WinForms window must have a distinct, traceable ASP.NET counterpart. This is a functional parity port, not a visual-only recreation.

### Reuse-first architecture

1. **Reuse `PKHeX.Core` unchanged** as the domain, file-format, legality, and Pokémon logic engine. Reference the existing project; do not duplicate or reimplement its behavior in the web layer.
2. **Reuse existing C# source files wherever they are platform-neutral.** Prefer project references or linked source files over copying files. Preserve their namespaces and behavior where practical.
3. **Keep WinForms-specific code in `PKHeX.WinForms`.** Do not try to compile `Form`, `Control`, `MessageBox`, `PropertyGrid`, Windows dialogs, drawing/UI event code, or other Windows-only dependencies into the cross-platform web application.
4. **Separate mixed UI and business logic only when necessary.** If a WinForms class contains reusable operations mixed with control/event code, extract the smallest platform-neutral service or class into a shared application layer. Both desktop and web interfaces should call that same implementation.
5. **Replace only the interface layer.** Convert forms, controls, dialogs, menus, layout, browser interaction, and UI-specific event wiring to ASP.NET Core/Razor equivalents. Preserve the underlying operations, validation rules, data types, and workflows.
6. **Do not duplicate business rules.** A behavior should have one shared implementation wherever feasible. If a platform-specific implementation is unavoidable, document why and add equivalence tests.
7. **Keep the original desktop application building and working** throughout the port. It remains a behavioral reference and regression target until web parity is verified.

Before creating projects or moving files, inspect the actual project references, target frameworks, namespaces, and dependency graph. Reuse the existing web host in the companion `pkhex-docker` repository if its architecture and licensing/integration constraints make it a suitable host; do not create a competing host without first evaluating it.

### Window parity

- Every user-facing WinForms window gets its own named ASP.NET counterpart.
- Do not merge unrelated desktop windows into one generic page merely to reduce page count.
- A window may be represented by a dedicated Razor Page, MVC endpoint/view, or browser modal when its interaction is genuinely dialog-sized. The mapping must be explicit and documented.
- WinForms UserControls map to reusable Razor components unless independently launched/user-facing.
- A desktop window's tabs, menus, context menus, commands, validation, keyboard actions, and state transitions are part of its parity scope.

## Source baseline

- Repository: `tatsuki626/PKHeX`
- Source project: `PKHeX.WinForms`
- Domain project: `PKHeX.Core`
- Inventory baseline: `master` at the time this plan was created.
- Initial tree scan found 369 C# source files under `PKHeX.WinForms`. This is a source-file count, **not** a count of windows. The authoritative window count must come from inspecting class declarations and designer files, not guessing from filenames.

## Required inventory

Create and maintain `inventory.csv` (or an equivalent machine-readable manifest) with one row per user-facing Form/window and linked rows for reusable controls and shared logic.

Required fields:

| Field | Meaning |
|---|---|
| SourceType | Form, dialog, UserControl, menu/context menu, shared class, or other |
| SourcePath | Exact source path in PKHeX.WinForms or shared project |
| SourceClass | Exact C# class name |
| LaunchPoint | Menu item, toolbar action, parent form, or caller |
| Purpose | User-visible responsibility |
| ControlsAndTabs | Every meaningful control, tab, grid, menu, and input |
| CommandsAndValidation | Actions, event handlers, validation, and error paths |
| ReuseAssessment | Reuse unchanged, link source, extract shared logic, or replace UI |
| Dependencies | PKHeX.Core and other project/platform dependencies |
| WebCounterpart | Exact Razor Page/route/component/modal, or shared service |
| StateAndPersistence | Save/session state, file effects, and persistence |
| Tests | Automated behavior/parity tests |
| Status | Not inventoried, assessed, designed, implemented, tested, parity-verified |

For every source class, record what can be reused and what must be converted. No window is complete while any meaningful control or operation is unaccounted for.

## Reuse and conversion rules

| Existing source | Required treatment |
|---|---|
| `PKHeX.Core` classes | Reuse unchanged by project reference |
| Platform-neutral WinForms-project classes | Reuse unchanged where dependencies permit; link source or move only when justified |
| Logic mixed with WinForms controls | Extract minimal shared logic; desktop and web call the same implementation |
| Form / editor form | Preserve operations; replace UI with its own web page or modal |
| Dialog | Replace with dedicated browser modal/dialog preserving result and cancel semantics |
| UserControl | Reuse underlying logic; implement a reusable Razor component |
| TabControl / TabPage | Browser tabs with equivalent content and state |
| DataGridView | Accessible web data table with equivalent sorting/filtering/editing |
| PropertyGrid | Typed property editor with equivalent values and validation |
| ContextMenuStrip / MenuStrip | Browser context menu, navigation, or command menu |
| WinForms event handlers | Translate UI wiring into web actions; keep business operations shared and testable |
| Windows-only drawing, clipboard, filesystem, or dialog APIs | Replace only the platform-specific adapter/interaction; preserve domain behavior |

## Completion gates

1. **Reuse assessment complete:** each relevant source class is classified as reusable unchanged, linkable, requiring minimal extraction, or UI/platform-specific.
2. **Inventory complete:** every Form/dialog and every user-facing launch point is catalogued.
3. **Design complete:** every window has a named route/page/modal and dependency/reuse mapping.
4. **Shared logic complete:** reusable operations have one implementation; any unavoidable divergence is documented.
5. **Implementation complete:** every control and operation has a working web equivalent.
6. **Behavior verified:** tests cover validation, state transitions, save mutations, cancellation, errors, and relevant edge cases.
7. **Regression verified:** existing WinForms build/tests remain healthy; shared logic tests pass.
8. **Parity verified:** representative workflows are compared against desktop behavior; no feature is silently dropped.
9. **Release ready:** Linux Docker build, health check, persistence, and end-to-end browser tests pass.

## Delivery sequence

The ASP.NET host and substantial feature set already exist in `tatsuki626/pkhex-docker`; do not restart the hosting work or rewrite existing screens. The active work is a completeness audit followed by targeted gap closure.

1. **Freeze the inventory baseline.** Record the exact PKHeX source commit and the web-host branch/commit being compared. Do not mix source revisions in one inventory.
2. **Build the authoritative source inventory.** Enumerate all C# type declarations under `PKHeX.WinForms`, deduplicate partial classes, classify Forms/dialogs/UserControls/components/helpers by actual base type, and capture source paths.
3. **Trace launch points and ownership.** Find menu/toolbar handlers, constructors, `Show`/`ShowDialog` calls, parent windows and other callers. Cross-check code-only forms and designer pairs.
4. **Inspect each window.** Record tabs, meaningful controls, commands, validation, cancel/error paths, state transitions, save effects, Core operations and platform dependencies.
5. **Map to the existing web host.** Link each source window to its exact existing route/page/modal/component or a concrete new task. Map reusable controls and shared logic. Mark each control/command as implemented, partial, missing, excluded or not applicable.
6. **Prioritize gaps.** Fix correctness issues first, then missing operations in existing screens, then genuinely missing windows. Do not duplicate an existing counterpart.
7. **Implement in small slices.** Reuse PKHeX.Core and platform-neutral C#; extract only the smallest shared logic where necessary. Update the inventory and add targeted tests with each slice.
8. **Verify behavior and real saves.** Cover serialization, checksums, validation, cancel/no-op, undo/redo, audit, persistence and representative desktop/Core equivalence. Keep real user saves private.
9. **Audit assets and platform boundaries.** Ensure no Windows-targeted WinForms/Drawing dependency leaks into the Linux host; verify sprites and previews.
10. **Validate deployment and release.** Test a fresh Docker deployment, persistence, auth, database indexing, backups and restore; run full local CI at integration/release gates.

The detailed execution checklist and exit criteria are maintained in `docs/plans/phase-17-completeness-audit.md` in the web-host repository. The machine-readable inventory is the source of truth for per-window status.

## Non-goals / safeguards

- No rewrite of PKHeX.Core or duplication of its domain logic.
- No blanket claim that all WinForms source files can be compiled unchanged in ASP.NET; Windows-specific types must remain isolated.
- No unnecessary file moves, mass renaming, or source duplication.
- No feature removal or consolidation of a desktop window without explicit approval.
- No placeholder controls in place of functional editors.
- No direct browser exposure of filesystem paths or unsafe server-side file operations.
- All uploaded save handling must preserve original files and use safe, isolated working copies/backups.
