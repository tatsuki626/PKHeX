# WinForms → ASP.NET Web Port

## Project decision

Every user-facing WinForms window must have a distinct, traceable ASP.NET counterpart. This is a functional parity port, not a visual-only recreation.

- Keep `PKHeX.WinForms` intact as the behavioral reference until parity is verified.
- Keep `PKHeX.Core` as the domain/format/legality engine; do not reimplement core Pokémon logic in the web layer.
- The web host must be cross-platform ASP.NET Core. Do not attempt to host WinForms or require a Windows desktop session in the Linux deployment.
- Do not merge unrelated desktop windows into one generic page merely to reduce page count.
- A window may be represented by a dedicated Razor Page, MVC endpoint/view, or browser modal when its interaction is genuinely dialog-sized. The mapping must be explicit and documented.
- WinForms UserControls are mapped to reusable Razor components unless they are independently launched/user-facing windows.
- A desktop window's tabs, menus, context menus, commands, validation, keyboard actions, and state transitions are part of its parity scope.

## Source baseline

- Repository: `tatsuki626/PKHeX`
- Source project: `PKHeX.WinForms`
- Domain project: `PKHeX.Core`
- Inventory baseline: `master` at the time this plan was created.
- Initial tree scan found 369 C# source files under `PKHeX.WinForms`. This is a source-file count, **not** a count of windows. The authoritative window count must come from inspecting class declarations and designer files, not guessing from filenames.

## Required inventory

Create and maintain `inventory.csv` (or an equivalent machine-readable manifest) with one row per user-facing Form/window and linked rows for reusable controls.

Required fields:

| Field | Meaning |
|---|---|
| SourceType | Form, dialog, UserControl, menu/context menu, or other |
| SourcePath | Exact source path in PKHeX.WinForms |
| SourceClass | Exact C# class name |
| LaunchPoint | Menu item, toolbar action, parent form, or caller |
| Purpose | User-visible responsibility |
| ControlsAndTabs | Every meaningful control, tab, grid, menu, and input |
| CommandsAndValidation | Actions, event handlers, validation, and error paths |
| WebCounterpart | Exact Razor Page/route/component/modal |
| CoreDependencies | PKHeX.Core APIs and data types used |
| StateAndPersistence | Save/session state, file effects, and persistence |
| Tests | Automated parity/behavior tests |
| Status | Not inventoried, inventoried, designed, implemented, tested, parity-verified |

No window is complete while any meaningful control or operation is unaccounted for. Use source links/line references where practical.

## Mapping rules

| WinForms source | ASP.NET counterpart |
|---|---|
| Main Form | Main application page/layout and its individually mapped child windows |
| Form / editor form | Dedicated page or dedicated browser modal, preserving its own identity |
| Dialog | Dedicated modal/dialog component with equivalent result and cancel semantics |
| UserControl | Reusable Razor component |
| TabControl / TabPage | Browser tab set with equivalent content and state |
| DataGridView | Accessible web data table with sorting/filtering/editing as applicable |
| PropertyGrid | Typed property editor with equivalent validation |
| ContextMenuStrip / MenuStrip | Browser context menu / navigation or command menu |
| Event handlers | Explicit application commands/services with tests |

## Completion gates

1. **Inventory complete:** every Form/dialog and every user-facing launch point is catalogued.
2. **Design complete:** every window has a named route/page/modal and dependency mapping.
3. **Implementation complete:** every control and operation has a working web equivalent.
4. **Behavior verified:** tests cover validation, state transitions, save mutations, cancellation, errors, and relevant edge cases.
5. **Parity verified:** representative workflows are compared against desktop behavior; no feature is silently dropped.
6. **Release ready:** Linux Docker build, health check, persistence, and end-to-end browser tests pass.

## Delivery sequence

1. Inventory all Forms, dialogs, launch points, controls, and commands.
2. Build the web shell, routing, session/save lifecycle, and shared Razor controls.
3. Port windows in dependency order, keeping one-to-one traceability in the manifest.
4. Add parity tests alongside each window rather than postponing testing.
5. Review every manifest row before declaring the port complete.

## Non-goals / safeguards

- No blanket claim of feature parity based on a compiling web shell.
- No removal or consolidation of a desktop window without explicit approval.
- No substitution of placeholder controls for functional editors.
- No direct browser exposure of filesystem paths or unsafe server-side file operations.
- All uploaded save handling must preserve original files and use safe, isolated working copies/backups.
