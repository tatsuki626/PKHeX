# WinForms inventory status

## Baseline scan

Repository tree: `tatsuki626/PKHeX`, branch `master`.

A recursive Git tree scan returned:

- 369 C# files under `PKHeX.WinForms/`
- 159 WinForms designer partial files (`*.Designer.cs`)
- 210 other C# files under the project

These are file counts only. They are **not** the number of distinct Forms or dialogs: some designer partials belong to UserControls, some classes may be code-only, and generated designer files are paired with implementation files.

## Inventory procedure

The next inventory pass must classify each designer/implementation pair by its actual declaration and base type:

- `Form` / `Form` subclass: distinct window; must map to its own page or modal.
- `UserControl`: reusable Razor component unless independently exposed as a window.
- `Component`, menu, context menu, or helper: map to the corresponding web interaction/service and link it to every owning window.
- Code-only window: discover through class inheritance and launch sites, not filename heuristics.

For each Form, inspect both the declaration and all launch sites (menu handlers, button handlers, constructors, and `Show`/`ShowDialog` calls). Then inspect its designer controls and implementation event handlers. A filename-only list is not an acceptable final inventory.

## Current status

| Work item | Status |
|---|---|
| Establish one-to-one parity rule | Complete |
| Create isolated web-port branch | Complete |
| Commit porting plan | Complete |
| Count WinForms C# and designer files | Complete |
| Classify all Forms vs UserControls | Not started |
| Enumerate every window's controls and commands | Not started |
| Map every window to a web route/modal | Not started |
| Implement and test counterparts | Not started |

Do not report the window inventory as complete until every Form and launch point has been classified and cross-checked.
