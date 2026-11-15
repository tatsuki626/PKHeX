# Architecture inspection: PKHeX source and existing web host

Inspection date: 2026-11-15

## Scope and method

Read the project files, project references, representative WinForms source, and the companion web repository tree and key hosting/build files through GitHub. Repositories inspected:

- `tatsuki626/PKHeX`, `master` (desktop source)
- `tatsuki626/pkhex-docker`, `experiment/upstream-winforms-web-port` (existing ASP.NET application)

This was a source inspection only. No local restore, build, Docker build, or runtime test was performed as part of this inspection.

## Findings: desktop repository

### Project compatibility

| Project | Target | Finding |
|---|---|---|
| `PKHeX.Core` | `net10.0` | Cross-platform class library; directly reusable from ASP.NET Core |
| `PKHeX.WinForms` | `net10.0-windows`, `UseWindowsForms=true` | Windows desktop UI; must not be referenced by the Linux web host |
| `PKHeX.Drawing` | `net10.0-windows`, System.Drawing.Common | Windows-targeted drawing dependency; not directly suitable for Linux web hosting |
| `PKHeX.Drawing.Misc` | `net10.0-windows` | Depends on Core and Windows-targeted drawing projects |
| `PKHeX.Drawing.PokeSprite` | `net10.0-windows` | Windows-targeted project; web sprite delivery needs a compatible asset/rendering path |
| `Tests/PKHeX.Core.Tests` | Core test project | Existing Core test suite should remain part of regression checks |

The WinForms project references Core plus all three Drawing projects. The web host should reference Core directly and should not take a project reference on WinForms or the Windows-targeted Drawing projects.

### What source inspection shows

- `PKHeX.WinForms/MainWindow/Main.cs` derives from `Form` and combines menus, file dialogs, drag/drop, plugins, save lifecycle, and calls into Core. It is a desktop composition root, not a reusable web class.
- `PKHeX.WinForms/Controls/PKM Editor/PKMEditor.cs` derives from `UserControl`; it owns WinForms controls, data binding, validation state, and editor events. Its UI must be replaced, while the PKM model operations and Core helpers it invokes should remain shared.
- `EditPK9.cs` illustrates the split: field loading/saving is tied to the editor control, while format-specific mutations such as `FixMoves`, `FixRelearn`, `FixMemories`, and checksum refresh are operations on Core entities. The web editor should use the same Core operations, not copy a second rules engine.
- `LoadSave.cs` directly reads/writes WinForms control state and cannot be linked wholesale into ASP.NET.
- `BatchEditor.cs` is a WinForms Form, but it delegates instruction processing to Core types such as `EntityBatchProcessor`, `EntityBatchEditor`, and slot abstractions. The web UI should call those same Core services/types and replace only the folder picker, grid, instruction editor, preview, and commit/cancel interaction.
- A helper such as `SortableBindingList<T>` is not a WinForms Form, but it is built around `BindingList<T>`/component-model presentation semantics. It should be assessed individually rather than assumed to be application-domain logic.

**Conclusion:** Most of the reusable domain logic is already correctly located in `PKHeX.Core`. Many WinForms files are interface adapters and should not be linked unchanged. There is no evidence supporting a blanket link of the entire WinForms project into the web app.

## Findings: existing `pkhex-docker` application

### It is already a viable ASP.NET host

The inspected branch contains a .NET 10 ASP.NET Core project (`PKHexWeb.csproj`) with:

- A minimal-API host in `Program.cs`
- Cookie authentication, first-run administrator setup, authorization, data protection, and login throttling
- Save working-copy storage, undo/redo history, backups, audit events, and safe read-only browsing of mounted save/database folders
- Modular API groups for the existing editor and tools phases
- A static browser client under `wwwroot/` (HTML, CSS, JavaScript modules and panels)
- Docker and Compose deployment, non-root runtime, health check, persistent `/data` volume, CI workflows, generated fixtures, smoke tests, and helper-oracle tests

The UI is currently plain HTML/CSS/JavaScript rather than Razor Pages. That does not prevent reuse: ASP.NET Core can continue serving the existing frontend and APIs while new screens are added incrementally. There is no need to replace the working host with a new ASP.NET project merely to satisfy the one-window/one-counterpart requirement.

### PKHeX source integration

- `PKHexWeb.csproj` references `vendor/PKHeX/PKHeX.Core/PKHeX.Core.csproj`, not a NuGet package and not the WinForms project.
- `scripts/fetch-pkhex-source.sh` fetches an immutable upstream source archive and verifies its SHA-256.
- The current pin is `3767c1367aeee2db93c2023d9938e9334e9d9819`.
- `Dockerfile` uses the pinned source, publishes the ASP.NET app, and retains the source archive and GPL licence in the image.
- `THIRD_PARTY_NOTICES.md` and the root licence record the GPL-3.0-or-later obligations.
- `Program.cs` directly consumes `PKHeX.Core` types including `SaveFile`, `SaveParser`, save interfaces, and `LegalityAnalysis`.

This is aligned with the reuse-first architecture. The source pin is deliberate and should remain explicit; it must be checked against the intended PKHeX fork/upstream baseline before any update.

### Existing feature work to preserve

The README and roadmap document substantial existing web functionality across save loading, box/party editing, Pokémon editing, legality, transfers, trainer/inventory/mail, Pokédex, Mystery Gifts/events, batch tools, Showdown, QR, reports, databases, generation-specific save editors, block/container tools, backups, audit, and deployment.

This is existing implementation, not proof that every WinForms window is already represented. The roadmap itself records partial/gap areas and some deliberate exclusions. We must map each upstream window to an existing web screen/API or a new dedicated counterpart and identify gaps. Existing self-hosting features should be preserved unless explicitly decided otherwise.

## Architecture decision recommended by this inspection

1. Use `tatsuki626/pkhex-docker` as the existing ASP.NET host and deployment foundation, subject to confirming its current branch/CI state and aligning the source pin.
2. Keep `tatsuki626/PKHeX` as the fork/source inventory and traceability reference. Do not create a competing web host in it.
3. Continue to reference the unmodified `PKHeX.Core` project from the selected, pinned PKHeX source revision.
4. Do not reference `PKHeX.WinForms`, `PKHeX.Drawing`, `PKHeX.Drawing.Misc`, or `PKHeX.Drawing.PokeSprite` from the Linux ASP.NET application while they target Windows.
5. Reuse Core APIs and algorithms directly. For any valuable WinForms-only logic, first check whether its operation already exists in Core. If not, extract the smallest UI-independent class into a cross-platform shared project only when extraction is necessary and safe for the desktop build.
6. Replace the WinForms view/event layer with browser UI. Preserve each Form's identity as its own route or dedicated modal; reusable UserControls become web components.
7. Keep the existing JavaScript UI if it is fit for purpose. Razor is an option, not a requirement; avoid a frontend rewrite that does not add parity or maintainability.
8. Preserve the existing safety, authentication, backups, undo/redo, audit, database, tests, CI, and Docker behavior.

## Important caveats / follow-up checks

- The inspected `pkhex-docker` branch has not been built or run during this inspection. Verify branch head, CI, Docker build, and smoke suite before selecting it as the integration base.
- The source pin in `pkhex-docker` is from 2026-11-07; compare it with the desired fork baseline and check for API drift before changing it.
- `PKHeX.Drawing*` are Windows-targeted. Sprite/image rendering must be solved separately for web (for example, serve compatible sprite assets or implement a cross-platform renderer); do not pull System.Drawing into the Linux web process by accident.
- The current web app's existing feature list must be cross-referenced against a complete Form/launch-point inventory. Feature overlap alone does not establish one-to-one UI parity.
- The full WinForms inventory and file-by-file reuse classification remain outstanding.

## Recommended next work

1. Verify the current branch head and CI/build state of `pkhex-docker`.
2. Produce a machine-readable WinForms inventory by class/base type and launch point.
3. Map every Form to an existing or new web route/modal and every UserControl to a reusable web component.
4. For each window, record its existing Core operations, missing operations, UI-only dependencies, and test coverage.
5. Implement the highest-value parity gaps on the existing host, with no duplicate domain logic.
