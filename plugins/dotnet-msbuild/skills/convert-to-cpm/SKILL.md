---
name: convert-to-cpm
description: Convert .NET projects and solutions to use NuGet Central Package Management (CPM). Use when the user wants to centralize, update, bump, upgrade, align, or sync NuGet package versions across multiple projects — including adopting Directory.Packages.props, removing per-project Version attributes, resolving version conflicts and mismatches, or getting package versions consistent and up to date across a repository or solution.
---

# Convert to Central Package Management

Migrate .NET projects from per-project package versioning to NuGet Central Package Management (CPM). CPM centralizes all package versions into a single `Directory.Packages.props` file, making version governance and upgrades easier across multi-project repositories.

## When to Use

- The user wants to adopt Central Package Management for a .NET repository, solution, or project
- Package versions are scattered across many `.csproj`, `.fsproj`, or `.vbproj` files and the user wants a single source of truth
- The user mentions `Directory.Packages.props`, CPM, or centralizing NuGet versions
- The user wants to update, bump, upgrade, align, or sync a NuGet package version across multiple projects — CPM is the recommended approach for managing shared package versions, so suggest converting to CPM as part of the update if the projects use `PackageReference` and CPM is not already enabled
- Package versions are out of sync, conflicting, or mismatched across projects and the user wants to resolve or unify them

## When Not to Use

- The repository already has CPM fully enabled for all in-scope projects
- The user is working with `packages.config`-based projects (must first migrate to `PackageReference`)
- The user wants to manage versions via a custom MSBuild property file without using CPM

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Scope | Yes | A project file, solution file, or directory containing .NET projects to convert |
| Version conflict strategy | No | How to resolve cases where the same package has different versions across projects (default: use highest version) |

## Workflow

### Step 1: Determine scope

- **Single project**: User specifies a `.csproj`, `.fsproj`, or `.vbproj`.
- **Solution**: User specifies a `.sln` or `.slnx`. List projects with `dotnet sln list`.
- **Repository/directory**: No specific file given. Find all project files recursively from the first common ancestor directory of all .NET projects in scope.

If the scope is unclear, ask the user.

### Step 2: Establish baseline build

Before making any changes, verify the scope builds successfully and capture a baseline binlog and package list. Run `dotnet clean`, then `dotnet build -bl:baseline.binlog`, then `dotnet list package --format json > baseline-packages.json`. See [baseline-comparison.md](references/baseline-comparison.md) for the full procedure and fallback options. If the baseline build fails, stop and inform the user — the scope must build cleanly before conversion.

### Step 3: Check for existing CPM

Search for any existing `Directory.Packages.props` in scope or ancestor directories. If CPM is already fully enabled, inform the user and stop. If a `Directory.Packages.props` exists without CPM enabled, ask whether to add the property to the existing file or create a new one.

### Step 4: Audit package references

Extract all `<PackageReference>` items and versions from in-scope project files. Also scan `<Import>` elements to discover shared `.props`/`.targets` files containing package references.

Check for complexities: version conflicts, MSBuild property-based versions, conditional references, security advisories, and existing `VersionOverride` usage. See [audit-complexities.md](references/audit-complexities.md) for the full checklist.

Present audit results to the user before proceeding, including:
- A table of each package, its version(s), and which projects use it
- Any version conflicts, security advisories, or complexities requiring decisions

When version conflicts exist, present each one individually with the affected projects, the distinct versions found, and the resolution options (align to highest, use `VersionOverride`, upgrade for security, etc.) with their trade-offs. Ask the user to decide on each conflict before proceeding. See [audit-complexities.md § Same package with different versions](references/audit-complexities.md) for the resolution workflow and presentation format.

### Step 5: Create or update Directory.Packages.props

Create the file with `dotnet new packagesprops` (.NET 8+) or manually. Add a `<PackageVersion>` entry for each unique package sorted alphabetically. For conditional versions or `VersionOverride` patterns, see [directory-packages-props.md](references/directory-packages-props.md).

### Step 6: Update project files

Remove the `Version` attribute from every `<PackageReference>` that now has a corresponding `<PackageVersion>`. Also update any shared `.props`/`.targets` files identified in step 4.

- Preserve all other attributes (`PrivateAssets`, `IncludeAssets`, `ExcludeAssets`, `GeneratePathProperty`, `Aliases`)
- Preserve conditional `<ItemGroup>` elements — only remove the `Version` attribute within them
- Retain each file's existing indentation style (spaces vs. tabs, indentation depth) and blank lines — do not reformat or reorganize unchanged lines
- Use `VersionOverride` (with user confirmation) when a project needs a different version than the central one

### Step 7: Handle MSBuild version properties

For `PackageReference` items that used MSBuild properties for versions, determine whether to inline the resolved value or keep the property reference in `Directory.Packages.props`. After validation succeeds in step 8, remove inlined version properties from `Directory.Build.props` or other files, verifying they have no remaining references. See [msbuild-property-handling.md](references/msbuild-property-handling.md) for the decision workflow, import order requirements, and cleanup procedure.

### Step 8: Restore and validate

Run a clean restore and build, capturing a post-conversion binlog and package list. Run `dotnet clean`, then `dotnet build -bl:after-cpm.binlog`, then `dotnet list package --format json > after-cpm-packages.json`. See [baseline-comparison.md](references/baseline-comparison.md) for the full procedure. If errors occur, see [validation-and-errors.md](references/validation-and-errors.md) for NuGet error codes and multi-TFM guidance.

### Step 9: Post-conversion report

Generate a `convert-to-cpm.md` markdown file alongside the binlog and JSON artifacts. This file should be self-contained and shareable — suitable for a pull request description, a team review, or a record of what was done. Structure the report with the following sections:

#### Section 1: Conversion overview

Summarize what was converted: the scope (project, solution, or repository), number of projects converted, total packages centralized, any projects or packages that were skipped, and any MSBuild properties that were inlined or removed. This gives the reader immediate context.

#### Section 2: Version conflict resolutions

If any version conflicts were encountered, list each one with:

- The package name and all versions that were found across projects
- Which projects used each version
- What the user decided (aligned to highest, used `VersionOverride`, upgraded for security, etc.)
- The practical impact: which projects now resolve a different version than before, and which are unchanged
- Any security advisories that influenced the decision

If no conflicts were found, state that all packages had consistent versions across projects — this is a positive signal worth noting.

#### Section 3: Package comparison — baseline vs. result

Compare `baseline-packages.json` and `after-cpm-packages.json` per project. See [baseline-comparison.md](references/baseline-comparison.md) for the comparison procedure. Present two tables:

- **Changes table**: Packages where the resolved version changed, a `VersionOverride` was introduced, or a package was added/removed. Include a status column explaining what changed and why (e.g., "⚠️ Upgraded from 8.0.4 → 10.0.1 (security fix)", "VersionOverride — project retains pinned version").
- **Unchanged table**: All other packages, confirming they resolve identically to baseline.

If there are no changes at all, state that the conversion is fully version-neutral — this is the ideal outcome and provides reassurance.

#### Section 4: Risk assessment

Provide a clear confidence statement:

- **✅ Low risk** — Conversion is version-neutral; all packages resolve to the same versions as baseline. The build and restore succeeded. Recommend running `dotnet test` as a final check.
- **⚠️ Moderate risk** — Some packages changed versions (e.g., minor/patch alignment or security upgrades). List the affected packages and projects. Recommend reviewing the changes table and running `dotnet test` to verify no regressions.
- **🔴 High risk** — Major version changes were applied, or packages were added/removed unexpectedly. Recommend careful review, running `dotnet test`, and comparing binlogs before merging.

Call out any specific warnings: security advisories on retained versions, `VersionOverride` usage that partially undermines centralization, or MSBuild property removal that could affect other build logic.

#### Section 5: Artifacts and next steps

List the artifacts produced during conversion and explain how to use them:

- **`baseline.binlog`** and **`after-cpm.binlog`** — MSBuild binary logs capturing the complete structured build event stream before and after conversion. These record every property evaluation, item resolution, target execution, and package reference resolution that MSBuild performed. Open them in the [MSBuild Structured Log Viewer](https://msbuildlog.com/) (`winget install KirillOsenkov.MSBuildStructuredLogViewer` on Windows) to inspect a searchable tree of the full build — including how each `PackageReference` was resolved, which properties contributed to version selection, and the complete dependency graph. Comparing the two binlogs lets the user verify exactly how package references and other MSBuild details are processed before and after conversion.
- **`baseline-packages.json`** and **`after-cpm-packages.json`** — Machine-readable snapshots of resolved package versions per project, used to produce the comparison tables above.
- **`convert-to-cpm.md`** — This report file, suitable for use as a pull request description or team review artifact.

Recommend the user run `dotnet test` to validate runtime behavior beyond build success. If any version conflicts were resolved by upgrading, recommend reviewing the release notes for the upgraded packages.

## Validation

- [ ] Baseline build succeeded before any changes were made
- [ ] `Directory.Packages.props` exists with `ManagePackageVersionsCentrally` set to `true`
- [ ] Every in-scope `PackageReference` either has no `Version` attribute or uses `VersionOverride`
- [ ] Every referenced package has a corresponding `PackageVersion` entry
- [ ] `dotnet restore` and `dotnet build` complete without errors from a clean state
- [ ] Package list comparison shows no unexpected version changes
- [ ] No orphaned version properties remain (unless intentionally kept)

## More Info

- [Central Package Management documentation](https://learn.microsoft.com/nuget/consume-packages/central-package-management)
- [Validation and common errors](references/validation-and-errors.md)
