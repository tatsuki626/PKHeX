# WinForms inventory status

## Scope and baseline

Desktop source repository: `tatsuki626/PKHeX`  
Desktop project: `PKHeX.WinForms`  
Web host: `tatsuki626/pkhex-docker`, branch `experiment/upstream-winforms-web-port`  
PKHeX.Core source pin used by the web host: `3767c1367aeee2db93c2023d9938e9334e9d9819`

The current recursive source-tree scan of desktop `master` at `9a1a8f4a450996e9572ee4428e9ae709166c2c6d` found:

- 369 C# source files under `PKHeX.WinForms/`
- 159 files named `*.Designer.cs`; 158 have matching implementation files, while `Properties/Resources.Designer.cs` is generated resources rather than a window designer
- 158 designer/implementation pairs
- 52 code-only C# files without a matching designer pair

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
| Generate complete type/declaration inventory | In progress: all 158 designer-paired implementation declarations inspected; 114 Forms and 44 UserControls recorded in provisional `inventory.csv`; code-only classes also scanned |
| Classify Forms, dialogs, UserControls and components | In progress: all 158 designer-paired declarations classified; 52 code-only files scanned for additional window declarations; linked helper/component inventory still being expanded |
| Trace every Form launch point and owning window | In progress: launch sites traced for 114/114 Forms; `ErrorWindow` and `SAV_Capture7GG` are both confirmed reachable |
| Enumerate meaningful controls, commands and validation | In progress: detailed semantic mapping started for Main, SAVEditor and PKMEditor; see [CORE-CONTROL-MAPPING.md](CORE-CONTROL-MAPPING.md) |
| Assess source reuse and Windows-only dependencies per type | Partial: initial reuse treatment recorded; dependency-level review pending |
| Map each Form to existing/new web counterpart | In progress: exact candidate web route/component mappings and probable gaps documented for Main, SAVEditor and PKMEditor |
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
- Recursive tree: 369 C# files, 159 designer files and 12,171 total tree entries. There are 158 designer/implementation pairs; the remaining designer file is generated resources.
- All 158 paired implementation files were inspected and classified by their primary declaration: 114 Forms and 44 UserControls. The 52 code-only files were also scanned; no additional Form declaration was found. This is the complete first-pass Form/UserControl classification for this exact source commit, subject to a launch-site cross-check.
- Created [`inventory.csv`](inventory.csv) with 158 Form/UserControl rows (114 Forms, 44 UserControls), and [`source-audit.csv`](source-audit.csv) with the 52 code-only files and their primary declarations. The inventory remains provisional because most owning windows, control details, shared helper relationships and web counterpart mappings are still pending.
- Verified main-window launch sites in `Main.cs` for About, ReportGrid, KChart, SAV_Database, SAV_Encounters, SAV_MysteryGiftDB, SettingsEditor, BoxExporter, BatchEditor, SAV_FolderList and QR.
- Important classification findings: `PKMEditor`, `BoxEditor`, `PartyEditor` and `SAVEditor` are UserControls, not independent Forms; `PokePreview`, `BallBrowser` and `StatusBrowser` are Forms despite residing under Controls; several Gen 4 Pokeathlon/Safari and Gen 5 Join Avenue classes are UserControls nested within larger editor workflows.
- The inventory records verified launch sites for the Main window, SAVEditor, PKMEditor and selected nested dialogs; remaining launch ownership, exact web counterpart mapping and control-level parity are pending. No parity completion claim is made from this seed.


The next inventory subtask is to inspect the two possible orphan Forms and then capture control/command inventories for each reachable Form, starting with the shared Main/SAVEditor workflows. Do not count a Form as out of scope until its references and launch paths have been checked.


## Core control-level pass (2026-11-17)

Created [`control-audit.csv`](control-audit.csv) from the desktop designer files for the three central surfaces:

| Surface | Designer fields | Explicit event bindings |
|---|---:|---:|
| Main | 44 | 34 |
| SAVEditor | 84 | 67 |
| PKMEditor | 218 | 115 |
| **Total** | **346** | **216** |

This is a source inventory, not a web parity result. The next review must classify each control as functional, decorative/layout, conditional by save capability, or dynamically created; map functional controls and commands to the web UI/API; and inspect validation/state/persistence semantics in the implementation handlers. The designer regex captures explicit method-group event bindings; dynamically registered events and inherited controls need a separate source review.


## First web counterpart mapping

Created [`window-mapping.csv`](window-mapping.csv) with feature-level mappings for Main, SAVEditor and PKMEditor. The mappings identify the existing browser shell, save workspace and schema-driven Pokémon editor, plus their known API route families. These are candidate counterparts only: menu/control-level mapping, capability rules, validation, error/cancel behavior and parity tests remain to be checked. No duplicate screen should be created before that comparison.


## Window reachability cross-check (2026-11-17)

- **ErrorWindow is not an orphan.** Its static `ShowErrorDialog` entry point is called by both `Program.UIThreadException` and `Program.CurrentDomain_UnhandledException` in non-DEBUG builds. The dialog exposes a friendly message, exception details, loaded assemblies, copy-details, continue and abort/exit behavior. It needs a web-host error-reporting equivalent; it must not be excluded from parity.
- **SAV_Capture7GG is reachable; it is not an orphan.** The launch chain is `SAVEditor.cs` (the `SAV7b` branch in the Pokédex dispatch) → `SAV_PokedexGG` → `SAV_Capture7GG`. `SAV_PokedexGG.cs:59` calls `new SAV_Capture7GG(SAV).ShowDialog()`. The dialog edits Let's Go per-species captured/transferred counts and aggregate totals, supports bulk setting and sum-total calculation, and commits changes to the origin save only on Save; Cancel closes without copying changes. The inventory row now records this nested launch chain and required web behavior.
- The inventory row for `ErrorWindow` now records its two production exception-handler call paths. No code was changed.


- The earlier absence finding was caused by searching only direct launchers in `Main.cs` and `SAVEditor.cs`. The actual dialog is launched one level deeper from `SAV_PokedexGG`; this demonstrates why nested Form ownership must be traced recursively rather than inferred from top-level menu dispatch alone.
- **Result: 0 confirmed orphan Forms among the two candidates checked.** Both are reachable and remain in the parity scope. All 114 Forms now have a traced launch/owner path in the first-pass inventory; this does not mean their web parity has been audited.
\n\n## Detailed central-surface control mapping (2026-11-17)\n\n- Created [CORE-CONTROL-MAPPING.md](CORE-CONTROL-MAPPING.md) with semantic maps for Main, SAVEditor and PKMEditor: menus, tabs, functional controls, event/handler families, web route/component candidates, format/capability rules and probable gaps. Added [control-mapping-detail.csv](control-mapping-detail.csv), a 346-row per-control mapping with source event bindings, functional grouping, web counterpart candidate and review status.\n- This is an evidence-backed working map, not parity sign-off. It distinguishes candidate counterparts from verified behavior and records where exact endpoint/handler/test comparison remains outstanding.\n- Main covers global file/tools/options commands, legality, drag/drop and editor event wiring. SAVEditor covers box/party/daycare/extra slots, save tools, generation-specific editor launchers and runtime slot/search/undo interactions. PKMEditor covers six editor tabs, dependent fields, validation, format gates, helper actions and nested editors.\n- Next: inspect exact web UI/endpoint implementation for each high-risk candidate and mark each control/command implemented, partial, missing, excluded or not applicable only when evidence supports it.\n