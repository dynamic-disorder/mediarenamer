# MediaRenamer — Working Instructions

| | |
| :--- | :--- |
| **Need** | A session with no memory of this repository: three projects, a Windows-only runtime that still cross-compiles on Linux, the metadata fallback chain, and the README that describes a directory layout the tree does not have. |
| **Why** | Every change here touches real media files. A wrong timestamp or a rename applied on the wrong pass is unrecoverable, and a build-flag mistake turns a working Linux build into `NETSDK1100`. |
| **What** | Project map, build and run split by platform, the metadata fallback chain, the rename plan's collision rule, `.agents/` configuration, and the drift already found. Deliberately not repeated: `TODOS.md`, which owns planned features. |
| **DoD** | A session can build on either platform, predict which timestamp a file will be renamed from, and know why a dry run is the only safe first move. |
| **Top risk** | Renaming a folder of originals without a dry run, or trusting the EXIF path for a format it does not cover, so the files end up named from a copy time. |

Loaded automatically as workspace instructions for anything under `mediarenamer/`.

## What this repository is

One solution, `MediaRenamer.sln`, with three `net10.0-windows` projects:

| Project | Assembly | Role |
| :--- | :--- | :--- |
| `Models/` | `MediaRenamer.Core.Models` | `MediaFileInfo`, `RenamePlanItem` — data only |
| `Services/` | `MediaRenamer.Core.Services` | `IMetadataProvider`, `ExifMetadataProvider`, `FileRenamer` |
| `WpfApp/` | `MediaRenamer.WpfApp` | WPF window, references both libraries |

The tool renames photos by capture time. It reads EXIF, falls back to filesystem time, builds a rename plan first, and only then applies it. There is **no test project** — that is a known gap in `TODOS.md`, not an oversight to fix silently in an unrelated change.

## Build and run

```bash
dotnet build MediaRenamer.sln                      # works on Linux and Windows
dotnet run --project WpfApp                        # Windows only
```

The split is the point: the build cross-compiles everywhere, the app runs only on Windows. `net10.0-windows` needs the targeting pack, and the XAML compiler runs cross-platform because the repository-root `Directory.Build.props` sets `EnableWindowsTargeting=true`. Removing that property breaks the build on Linux and macOS with `NETSDK1100`.

`Models` and `Services` set `UseWPF` even though they are libraries. That is deliberate — they use WPF-dependent types — and clearing it will not compile.

Where `$HOME` is not writable — the agent sandbox — source `local_dev/scripts/sandbox-env.sh` before restoring, or the first restore fails on a read-only NuGet cache.

## The timestamp chain

Each item's name comes from `MediaFileInfo.GetBestTimestamp()`, which is `MetadataTimestamp ?? FileCreationTime`. Four things follow:

- **Only images are read.** `ExifMetadataProvider` accepts `.jpg`, `.jpeg`, `.tif`, `.tiff` and `.png`; anything else returns `null` immediately.
- **The window scans more than the provider reads.** `Scan` filters in `.mp4` and `.mov` as well, but nothing extracts their metadata, so those files are always named from filesystem time.
- **A file with no EXIF silently falls back to `FileCreationTime`.** The provider wraps the whole read in `catch { return null; }`, so a corrupt file and a file without EXIF are indistinguishable to the caller — both get the fallback, and the plan's `Reason` records which one was used.
- **On Linux, "creation time" is the inode birth time**, which is often the copy time, not the capture time. Naming a folder on Linux therefore produces different results than the same folder on Windows.

Video metadata is not implemented. `TODOS.md` and `README.md` describe a `CompositeMetadataProvider` + ffprobe design; nothing in the tree implements it. When adding one, implement `IMetadataProvider` and compose it with EXIF rather than editing `FileRenamer`.

## The rename plan

`FileRenamer.BuildRenamePlan(files, pattern = "yyyyMMdd_HHmmss")` returns `RenamePlanItem`s and touches no files. `ApplyRename(plan, dryRun)` then moves them.

- The name is `timestamp + lowercased extension`, formatted with `CultureInfo.InvariantCulture` — keep the culture argument, or a machine with a non-Gregorian locale changes the output.
- **Collisions are resolved against the filesystem, not against the plan.** The `_1` suffix loop tests `File.Exists`, but `BuildRenamePlan` moves nothing, so two files that share a timestamp both receive the same target name. `ApplyRename` then moves the first and silently skips the second, because its target now exists. A folder of photos burst-shot inside one second loses names without an error — a real defect, not a design choice.
- `ApplyRename` skips a missing source and an existing target without reporting either. A partially applied batch looks finished; verify by listing the folder, not by trusting the status text.

`Scan` rebuilds the plan into a grid, and `Rename` applies exactly what the grid holds, with the dry-run checkbox deciding whether anything moves. Never add a code path that renames without going through that plan.

## Agent surfaces

| Path | Read by | Holds |
| :--- | :--- | :--- |
| `AGENTS.md` | Any agent | This file |
| `.agents/skills/` | Any agent | Seven tracked skills |
| `.agents/hooks/` | Git hooks | The commit gate |

The commit gate is the same one the sibling repositories use: `core.hooksPath` is `.githooks`, the wrapper execs `.agents/hooks/pre-commit-build-check.sh`. It builds the solution when staged `.cs`/`.csproj`/`.props`/`.targets` files exist, retries once, and exits 2 to block the commit. Both scripts must keep their `100755` index mode; `100644` makes a Linux clone skip the gate silently.

## Verification

```bash
dotnet build MediaRenamer.sln    # exit 0, every error with file and line
bash -n .agents/hooks/pre-commit-build-check.sh
```

There is no automated test, so evidence for a behaviour change is a dry run against a scratch folder:

```bash
# build a plan only, never apply it
dotnet run --project WpfApp
```

On Linux only the build is provable; the window needs Windows. State which platform a claim came from.

## Known drift — do not rediscover these

| Claim or symptom | Reality | Action |
| :--- | :--- | :--- |
| `README.md` shows `MediaRenamer.Core/` and `MediaRenamer.Wpf/` folders | The tree has `Models/`, `Services/`, `WpfApp/`; the assemblies carry the `MediaRenamer.Core.*` names, which is where the confusion started | Follow the tree, not the README's diagram |
| `README.md` describes `[Inject]` and `CompositeMetadataProvider` | Neither exists; the tree is plain constructor injection | Treat it as a design sketch |
| The tool "preserves creation time" | It names files from EXIF where available and falls back to filesystem time; `File.Move` preserves the file's timestamps but the tool does not set them | Describe what the code does |
| `TODOS.md` lists tests, video support, batch processing | All still absent — do not assume any of them exists | Read the code |

## Where work is recorded

Planned features live in `TODOS.md` and are the owner's list; adding to it is fine, silently implementing from it is not. Investigation output goes to `local_user_files/`, which is git-ignored except for its README.

Documentation: this file is the agent entry point. The per-tool files (`.clinerules`, `.claudeignore`, `.github/copilot-instructions.md`, `.agents/settings.json`) were removed on 2026-11-11 so nothing duplicates it.
