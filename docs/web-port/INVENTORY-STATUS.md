# WinForms inventory status

## Baseline scan

Repository tree: `tatsuki626/PKHeX`, branch `master`.

A recursive Git tree scan returned:

- 369 C# source files under `PKHeX.WinForms/`
- 159 WinForms designer partial files (`*.Designer.cs`)
- 210 other C# files under the project

These are file counts only. They are **not** the number of distinct Forms or dialogs: some designer partials belong to UserControls, some classes may be code-only, and generated designer files are paired with implementation files.

## Architecture inspection findings (2026-11-15)

See [ARCHITECTURE-INSPECTION.md](ARCHITECTURE-INSPECTION.md).

- `PKHeX.Core` targets `net10.0` and is cross-platform: reuse it directly.
- `PKHeX.WinForms` and all three `PKHeX.Drawing*` projects target `net10.0-windows`; do not reference them from the Linux ASP.NET host.
- The existing `tatsuki626/pkhex-docker` branch already provides a .NET 10 ASP.NET host, modular APIs, browser frontend, save lifecycle, auth, backups, undo/redo, audit, Docker, CI, and tests.
- Recommended: extend that existing host rather than create a competing web host. Its current PKHeX.Core source pin is `3767c1367aeee2db93c2023d9938e9334e9d9819`.
- Representative source review confirms that UI-heavy files such as `Main.cs`, `PKMEditor.cs`, and `BatchEditor.cs` cannot be linked wholesale, but they already call reusable Core operations. Replace their UI and keep those operations shared.
- The full class/window inventory and one-to-one mapping are still outstanding. No build or runtime test was run during this source-only inspection.

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
| Inspect Core/WinForms/Drawing target frameworks and project references | Complete |
| Inspect representative WinForms UI/Core coupling | Complete |
| Inspect existing ASP.NET host architecture and source pin | Complete |
| Commit architecture inspection report | In progress |
| Verify existing web-host branch head, CI, and Docker build | Not started |
| Classify all Forms vs UserControls | Not started |
| Enumerate every window's controls and commands | Not started |
| Map every window and shared service to web counterparts | Not started |
| Implement and test counterparts | Not started |

Do not report the window inventory or reuse assessment as complete until source declarations, dependencies, and launch points have been classified and cross-checked.
