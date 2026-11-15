# WinForms inventory status

## Baseline scan

Repository tree: `tatsuki626/PKHeX`, branch `master`.

A recursive Git tree scan returned:

- 369 C# source files under `PKHeX.WinForms/`
- 159 WinForms designer partial files (`*.Designer.cs`)
- 210 other C# files under the project

These are file counts only. They are **not** the number of distinct Forms or dialogs: some designer partials belong to UserControls, some classes may be code-only, and generated designer files are paired with implementation files.

## Reuse-first assessment procedure

For each source class, inspect its declaration, base type, namespaces, project references, and dependencies. Classify it as one of:

- **Reuse unchanged:** platform-neutral C# that can be consumed directly.
- **Link source:** platform-neutral source currently housed in the WinForms project but reusable through a linked compile item or shared project.
- **Extract minimal shared logic:** business operations mixed with WinForms controls; isolate only the reusable logic and make both desktop and web call it.
- **Replace UI/platform adapter:** code that depends on Windows Forms, Windows-only APIs, or desktop interaction.

Do not assume a whole file is reusable merely because it contains useful logic. Inspect dependencies first. Avoid copying or reimplementing existing behavior.

Classify designer/implementation pairs by actual declaration and base type:

- `Form` / `Form` subclass: distinct window; must map to its own page or modal.
- `UserControl`: reusable Razor component unless independently exposed as a window.
- `Component`, menu, context menu, or helper: map to the corresponding web interaction/service and link it to every owning window.
- Code-only window: discover through class inheritance and launch sites, not filename heuristics.

For each Form, inspect its launch sites (menu handlers, button handlers, constructors, and `Show`/`ShowDialog` calls), designer controls, implementation event handlers, and the operations those handlers invoke.

## Current status

| Work item | Status |
|---|---|
| Establish one-to-one window parity rule | Complete |
| Establish reuse-first architecture and conversion rules | Complete |
| Create isolated web-port branch | Complete |
| Commit initial porting plan | Complete |
| Count WinForms C# and designer files | Complete |
| Inspect project references and classify reusable classes | Not started |
| Evaluate companion `pkhex-docker` ASP.NET host for reuse | Not started |
| Classify all Forms vs UserControls | Not started |
| Enumerate every window's controls and commands | Not started |
| Map every window and shared service to web counterparts | Not started |
| Implement and test counterparts | Not started |

Do not report the window inventory or reuse assessment as complete until source declarations, dependencies, and launch points have been classified and cross-checked.
