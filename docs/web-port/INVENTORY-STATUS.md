# WinForms inventory status

## Scope and baseline

Desktop source repository: `tatsuki626/PKHeX`  
Desktop project: `PKHeX.WinForms`  
Web host: `tatsuki626/pkhex-docker`, branch `experiment/upstream-winforms-web-port`  
PKHeX.Core source pin used by the web host: `3767c1367aeee2db93c2023d9938e9334e9d9819`

The current recursive source-tree scan of desktop `master` at `9a1a8f4a450996e9572ee4428e9ae709166c2c6d` found:

- 369 C# source files under `PKHeX.WinForms/`
- 159 WinForms designer partial files (`*.Designer.cs`)
- 210 other C# files under the project

These are file counts, not counts of Forms or dialogs. Partial declarations must be deduplicated by fully qualified type; UserControls and components must not be counted as windows.

## Architecture findings

See [ARCHITECTURE-INSPECTION.md](ARCHITECTURE-INSPECTION.md).

- `PKHeX.Core` targets `net10.0` and is cross-platform; reuse it directly.
- `PKHeX.WinForms` and the `PKHeX.Drawing*` projects target Windows and must not be referenced by the Linux ASP.NET host.
- The existing `pkhex-docker` application is the chosen host. Its frontend is currently HTML/CSS/JavaScript with ASP.NET Core APIs; a Razor rewrite is not required.
- UI-heavy classes such as `Main`, `PKMEditor` and `BatchEditor` are not reusable wholesale, but their PKHeX.Core operations should remain shared.
- The latest reported local CI run for the web-host branch passed on 2026-11-17: Release build (0 warnings/errors), database tests, all 2,489 Living Dex legality checks, 2,243 end-to-end smoke checks and Docker image build. Report: `/mnt/Dropbox/pkhex/pkhex-local-ci-20261117-070239.md`.
- This CI result validates the current automated suite; it does not prove one-to-one WinForms parity or a fresh deployment's runtime behavior.

## Classification rules

For every declared type under `PKHeX.WinForms`, inspect its namespace, base type, designer pair, callers and dependencies. Assign one primary classification:

- **Form/dialog:** independently user-facing window; must map to a dedicated web page or dialog-sized modal.
- **UserControl:** reusable web component unless independently exposed as a window.
- **Component/menu/context menu:** map to the owning web interaction and record every owner.
- **Shared/helper logic:** assess for unchanged reuse, source linking, or minimal extraction.
- **Platform adapter:** replace Windows Forms, drawing, clipboard, filesystem-dialog or other desktop-only interaction.
- **Other:** explain why it is outside the user-facing parity scope.

Reuse treatment must be one of: reuse PKHeX.Core unchanged; reuse/link platform-neutral C# unchanged; extract the smallest shared logic; replace UI/platform adapter; or excluded with an explicit reason.

Do not classify by filename alone. Find code-only forms by inheritance and launch sites. Designer partials must be associated with their logical type.

## Inventory fields

The machine-readable inventory must record at least:

- stable ID and source commit;
- source type, path, fully qualified class and base type;
- designer path(s);
- launch point(s), owner(s) and caller evidence;
- purpose, tabs and meaningful controls;
- commands, event handlers, validation, cancel/error behavior;
- Core operations and other dependencies;
- reuse treatment and Windows-only dependencies;
- exact web counterpart (route/page/modal/component/service);
- per-control/command implementation status;
- state/persistence and save mutation effects;
- automated tests and real-save coverage;
- exclusions/limitations and evidence links;
- overall status: not inventoried, classified, mapped, partial, implemented, tested, parity-verified.

A window cannot be marked parity-verified while any meaningful control or command is unmapped or untested.

## Current status

| Work item | Status |
|---|---|
| Reuse-first architecture and no-Core-rewrite decision | Complete |
| Inspect Core/WinForms/Drawing target frameworks and project references | Complete |
| Inspect representative WinForms/Core coupling | Complete |
| Select existing ASP.NET host and document source pin | Complete |
| Establish one-to-one window parity rule | Complete |
| Update delivery sequence for inventory-first completion | Complete |
| Record Phase 17 audit, parity and release workstreams in web roadmap | Complete |
| Freeze exact desktop inventory commit | Complete: `9a1a8f4a450996e9572ee4428e9ae709166c2c6d` |
| Generate complete type/declaration inventory | In progress: provisional `inventory.csv` contains 113 inspected Form declarations and 42 inspected UserControl declarations; not exhaustive |
| Classify Forms, dialogs, UserControls and components | In progress: first 155 declaration rows classified; remaining source files and code-only types pending |
| Trace every Form launch point and owning window | In progress: 11 Main.cs launch points verified; all other owners/callers pending |
| Enumerate meaningful controls, commands and validation | Not started |
| Assess source reuse and Windows-only dependencies per type | Partial: initial reuse treatment recorded; dependency-level review pending |
| Map each Form to existing/new web counterpart | Not started |
| Prioritize and implement identified parity gaps | Pending inventory |
| Expand real-save and desktop-equivalence coverage | In progress |
| Audit web graphics/assets and Linux dependency boundaries | Pending |
| Validate fresh Docker deployment and persistent state | Pending |
| Complete final one-to-one parity audit | Pending |

## Immediate next task

Generate the authoritative type inventory from the pinned desktop source tree. The first pass should be mechanical and reproducible:

1. Obtain the recursive Git tree for the chosen source commit.
2. Enumerate all `.cs` files under `PKHeX.WinForms/`.
3. Parse namespace, type declaration, base type and partial modifier; associate designer files.
4. Deduplicate partial declarations by fully qualified type.
5. Identify candidate Form/dialog/UserControl types from inheritance, not file naming.
6. Save a provisional CSV/JSON manifest with evidence paths.
7. Inspect launch sites and confirm the candidate window list before calling it authoritative.

Do not start implementing new screens until the inventory and existing-web mapping show a real gap. Targeted correctness fixes discovered during inventory may proceed independently.



## First inventory pass (2026-11-17)

- Frozen desktop baseline: `master` at `9a1a8f4a450996e9572ee4428e9ae709166c2c6d`.
- Recursive tree: 369 C# files, 159 designer files; 12171 total tree entries.
- Created [`inventory.csv`](inventory.csv), a provisional seed with 113 directly inspected Form declarations and 42 directly inspected UserControl declarations. This is not the final Form count; other source files, code-only windows and designer partials remain to be classified.
- Verified main-window launch sites in `Main.cs` for About, ReportGrid, KChart, SAV_Database, SAV_Encounters, SAV_MysteryGiftDB, SettingsEditor, BoxExporter, BatchEditor, SAV_FolderList and QR.
- Examples of important classification findings: `PKMEditor`, `BoxEditor`, `PartyEditor` and `SAVEditor` are UserControls, not independent Forms; `PokePreview`, `BallBrowser` and `StatusBrowser` are Forms despite residing under Controls; several Gen 4 Pokeathlon/Safari and Gen 5 Join Avenue classes are UserControls nested within larger editor workflows.
- The CSV intentionally marks exact web counterpart mapping, control-level parity and full launch ownership as pending. No parity completion claim is made from this seed.
