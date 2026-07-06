# Packastack — Architecture & Developer Guide

## Table of Contents

1. [Overview](#1-overview)
2. [Project Layout](#2-project-layout)
3. [Entry Points & CLI](#3-entry-points--cli)
4. [Commands Reference](#4-commands-reference)
    - [init](#41-init)
    - [plan](#42-plan)
    - [build](#43-build)
    - [build-all](#44-build-all)
    - [build-subsets](#45-build-subsets)
    - [build-rc](#46-build-rc)
    - [refresh](#47-refresh)
    - [clean](#48-clean)
    - [completion](#49-completion)

5. [Architecture Layers](#5-architecture-layers)
    - [Core Layer](#51-core-layer-packastackcore)
    - [Target Layer](#52-target-layer-packastacktarget)
    - [Upstream Layer](#53-upstream-layer-packastackupstream)
    - [Planning Layer](#54-planning-layer-packastackplanning)
    - [Build Layer](#55-build-layer-packastackbuild)
    - [Debpkg Layer](#56-debpkg-layer-packastackdebpkg)
    - [APT Layer](#57-apt-layer-packastackapt)
    - [Logs Layer](#58-logs-layer-packastacklogs)
    - [AI Layer](#59-ai-layer-packastackai)

6. [Data Model](#6-data-model)
    - [Upstreams Registry](#61-upstreams-registry-upstreamsyaml)
    - [Build Type Selection](#62-build-type-selection)
    - [Dependency Graph](#63-dependency-graph)
    - [Build-All State Machine](#64-build-all-state-machine)

7. [Key Workflows](#7-key-workflows)
    - [Single Package Build Flow](#71-single-package-build-flow)
    - [Build-All Orchestration Flow](#72-build-all-orchestration-flow)
    - [Plan Command Flow](#73-plan-command-flow)

8. [Configuration](#8-configuration)
9. [Testing](#9-testing)
10. [Development Workflow](#10-development-workflow)

---

## 1. Overview

Packastack is a Python CLI tool for building OpenStack packages for Ubuntu. It automates the complete lifecycle of discovering, planning, and building Debian source and binary packages for OpenStack projects on Ubuntu.

**Key capabilities:**

- Resolve upstream sources for OpenStack projects via a declarative registry
- Determine build types (release tarballs vs. git snapshots)
- Fetch Ubuntu packaging repositories from Launchpad
- Compute dependency graphs and build orders
- Execute builds using git-buildpackage (gbp) and sbuild
- Produce source packages, binary packages, and build artifacts
- AI-powered build failure diagnosis and auto-fix via a pluggable skill system (build-doctor)

**Tech stack:** Python 3.14+, Typer CLI, GitPython, python-debian, launchpadlib, requests, Rich

---

## 2. Project Layout

```text
src/packastack/
  cli.py                   # Typer app definition, command registration
  __main__.py              # python -m entry point
  __init__.py              # Version metadata (__version__ = "0.1.0")

  core/                    # Configuration, paths, context objects, exceptions
    config.py              # YAML config load/save/merge
    context.py             # Immutable request/config dataclasses + mutable contexts
    paths.py               # Path resolution and directory creation
    run.py                 # RunContext: per-command logging/scoping
    exceptions.py          # Typed exceptions with exit codes
    spinner.py             # Rich-based activity spinner
    duration.py            # Duration parsing ("30m", "2h", "1d")

  target/                  # Ubuntu series/architecture resolution
    series.py              # resolve_series() — "devel" → actual codename
    arch.py                # Debian architecture detection (host arch → amd64, etc.)
    distro_info.py         # Parse /usr/share/distro-info/ubuntu.csv
    resolution.py          # Target expression parser (exact/prefix/contains/glob)

  upstream/                # Upstream source management
    registry.py            # UpstreamsRegistry: load/resolve upstreams.yaml
    source.py              # Tarball download, signature verification
    releases.py            # openstack/releases YAML parsing (versions, models)
    retirement.py          # Retirement detection from project-config + releases
    gitfetch.py            # GitFetcher: clone ubuntu-openstack-dev repos from LP
    tarball_cache.py       # Tarball caching with TTL
    pkg_scripts.py         # Package script helpers

  planning/                # Dependency resolution and build planning
    graph.py               # DependencyGraph: DAG of package deps
    graph_builder.py       # Build graph from debian/control files
    type_selection.py      # BuildType: RELEASE vs SNAPSHOT auto-selection
    package_discovery.py   # Discover packages from packaging repositories
    dependency_satisfaction.py  # Check if deps are satisfiable
    cycle_suggestions.py   # Suggest edges to break cycles
    deploop.py             # Debian loop dependency handling
    build_manifest.py      # Build manifest generation
    build_all_state.py     # BuildAllState: state machine for build-all
    validated_plan.py      # Plan validation
    targets.py             # Target resolution for planning
    control_min_versions.py # Minimum version extraction from control files

  build/                   # Build execution engine
    __init__.py            # Re-exports all public API
    types.py               # ResolvedTargets, WorkspacePaths, BuildInputs, BuildOutcome
    errors.py              # Exit codes, phase_error(), phase_warning()
    mode.py                # BuildMode: source/binary, sbuild/dpkg
    provenance.py          # BuildProvenance: audit trail for each build
    phases.py              # Phase functions: retirement check, registry resolution, etc.
    single_build.py        # Complete single-package build pipeline (all phases)
    all_runner.py          # Build-all orchestration (sequential + parallel)
    all_helpers.py         # Helper functions for build-all
    runner.py              # Unified build runner interface
    tarball.py             # Tarball acquisition (uscan + fallbacks)
    git_helpers.py         # Git commit, GPG, .gitattributes, version extraction
    localrepo_helpers.py   # Local APT repo helpers
    sbuild.py              # Sbuild wrapper with chroot bind-mounts
    sbuildrc.py            # Sbuild config discovery
    schroot.py             # Schroot management
    collector.py           # Artifact/log collection from sbuild output
    type_resolution.py     # Build type resolution
    tools.py               # Tool availability checks

  debpkg/                  # Debian packaging utilities
    control.py             # debian/control parser, ParsedDependency
    changelog.py           # debian/changelog parsing
    version.py             # ParsedVersion: epoch+upstream+debian_revision
    gbp.py                 # gbp patch-queue, build, PatchHealthReport
    gbpconf.py             # gbp.conf management
    watch.py               # debian/watch parser, uscan integration
    rules.py               # debian/rules patching (doctrees, sphinxdoc)
    launchpad_yaml.py      # launchpad.yaml parsing
    manpages.py            # Manpage handling
    dep_sync.py            # Dependency synchronization
    sudoers.py             # Sudoers configuration

  apt/                     # APT/Ubuntu archive interactions
    archive.py             # ArchiveFetcher: HTTP conditional fetch of Packages.gz
    localrepo.py           # Local APT repository (publish .deb, regenerate indexes)
    packages.py            # PackageIndex: merged snapshot + cloud-archive indexes

  logs/                    # Report generation and visualization
    plan_graph.py          # PlanGraph: DOT/ASCII/HTML/JSON renderers
    plan_reports.py        # Plan command report helpers
    type_selection.py      # Type selection reporting
    watch_resolution.py    # Watch file resolution reports
    dep_sync.py            # Dependency sync reports
    deps_satisfaction.py   # Dependency satisfaction reports
    all_reports.py         # Build-all summary reports

  ai/                      # AI-powered failure diagnosis (build-doctor skill system)
    client.py              # OpenAI-compatible API client
    skills.py              # Skill loader: reads skills/<name>/SKILL.md + YAML frontmatter
    runner.py              # run_skill(): single entry point for invoking an AI skill
    collectors.py          # Context collector registry (log/control/rules extractors)
    contracts.py           # Output contract parsers for skill responses
    triggers.py            # Cheap regex-based skill routing from log patterns
    build_diagnosis.py     # Build failure analysis → routes to skills, applies patches
    patch_diagnosis.py     # Patch failure analysis → drop/refresh upstreamed patches
    prompts.py             # Shared prompt helpers / skill-menu expansion
    memory.py              # AI memory persistence (ai-memory.json per run)

  skills/                  # AI skill definitions (markdown + YAML frontmatter, no Python)
    build-doctor/SKILL.md      # Router skill: picks the specialist for a failure
    build-patch/SKILL.md       # Generic patch author; safe fallback
    patch-diagnosis/SKILL.md   # Decide whether a quilt patch can be dropped
    patch-refresh/SKILL.md     # Refresh fuzzy patches after upstream drift
    patch-correction/SKILL.md  # Fix a patch that failed git apply --check
    python-compat/SKILL.md     # Interpreter bumps (distutils/imp/cgi removed, etc.)
    library-sync-advisor/SKILL.md  # Detect stale oslo/python-*client deps (guidance)

  commands/                # CLI command implementations
    init.py                # packastack init (tool pre-flight, idempotent metadata)
    plan.py                # packastack plan
    build.py               # packastack build (single, --all, and subset entry)
    build_rc.py            # packastack build rc / rc1 / rc2 ...
    build_subset.py        # Subset build helper (libraries / clients / services)
    build_helpers/         # Build helper utilities
    refresh.py             # packastack refresh ubuntu-archive
    clean.py               # packastack clean
    completion.py          # Shell completion handler

  data/
    upstreams.yaml         # Upstream registry (defaults + project overrides)
```

  upstream/                # Upstream source management
    registry.py            # UpstreamsRegistry: load/resolve upstreams.yaml
    source.py              # Tarball download, signature verification
    releases.py            # openstack/releases YAML parsing (versions, models)
    retirement.py          # Retirement detection from project-config + releases
    gitfetch.py            # GitFetcher: clone ubuntu-openstack-dev repos from LP
    tarball_cache.py       # Tarball caching with TTL
    pkg_scripts.py         # Package script helpers

  planning/                # Dependency resolution and build planning
    graph.py               # DependencyGraph: DAG of package deps
    graph_builder.py       # Build graph from debian/control files
    type_selection.py      # BuildType: RELEASE vs SNAPSHOT auto-selection
    package_discovery.py   # Discover packages from packaging repositories
    dependency_satisfaction.py  # Check if deps are satisfiable
    cycle_suggestions.py   # Suggest edges to break cycles
    deploop.py             # Debian loop dependency handling
    build_manifest.py      # Build manifest generation
    build_all_state.py     # BuildAllState: state machine for build-all
    validated_plan.py      # Plan validation
    targets.py             # Target resolution for planning
    control_min_versions.py # Minimum version extraction from control files

  build/                   # Build execution engine
    __init__.py            # Re-exports all public API
    types.py               # ResolvedTargets, WorkspacePaths, BuildInputs, BuildOutcome
    errors.py              # Exit codes, phase_error(), phase_warning()
    mode.py                # BuildMode: source/binary, sbuild/dpkg
    provenance.py          # BuildProvenance: audit trail for each build
    phases.py              # Phase functions: retirement check, registry resolution, etc.
    single_build.py        # Complete single-package build pipeline (all phases)
    all_runner.py          # Build-all orchestration (sequential + parallel)
    all_helpers.py         # Helper functions for build-all
    runner.py              # Unified build runner interface
    tarball.py             # Tarball acquisition (uscan + fallbacks)
    git_helpers.py         # Git commit, GPG, .gitattributes, version extraction
    localrepo_helpers.py   # Local APT repo helpers
    sbuild.py              # Sbuild wrapper with chroot bind-mounts
    sbuildrc.py            # Sbuild config discovery
    schroot.py             # Schroot management
    collector.py           # Artifact/log collection from sbuild output
    type_resolution.py     # Build type resolution
    tools.py               # Tool availability checks

  debpkg/                  # Debian packaging utilities
    control.py             # debian/control parser, ParsedDependency
    changelog.py           # debian/changelog parsing
    version.py             # ParsedVersion: epoch+upstream+debian_revision
    gbp.py                 # gbp patch-queue, build, PatchHealthReport
    gbpconf.py             # gbp.conf management
    watch.py               # debian/watch parser, uscan integration
    rules.py               # debian/rules patching (doctrees, sphinxdoc)
    launchpad_yaml.py      # launchpad.yaml parsing
    manpages.py            # Manpage handling
    dep_sync.py            # Dependency synchronization
    sudoers.py             # Sudoers configuration

  apt/                     # APT/Ubuntu archive interactions
    archive.py             # ArchiveFetcher: HTTP conditional fetch of Packages.gz
    localrepo.py           # Local APT repository (publish .deb, regenerate indexes)
    packages.py            # PackageIndex: merged snapshot + cloud-archive indexes

  logs/                    # Report generation and visualization
    plan_graph.py          # PlanGraph: DOT/ASCII/HTML/JSON renderers
    type_selection.py      # Type selection reporting
    watch_resolution.py    # Watch file resolution reports
    explain.py             # Explain command reports
    dep_sync.py            # Dependency sync reports
    deps_satisfaction.py   # Dependency satisfaction reports
    all_reports.py         # Build-all summary reports

  ai/                      # AI-powered failure diagnosis
    client.py              # OpenAI-compatible API client
    build_diagnosis.py     # Build failure analysis and auto-fix
    patch_diagnosis.py     # Patch failure analysis and refresh
    prompts.py             # Prompt templates for AI diagnosis
    memory.py              # AI memory persistence

  commands/                # CLI command implementations
    init.py                # packastack init
    plan.py                # packastack plan
    build.py               # packastack build
    build_rc.py            # packastack build-rc
    build_subset.py        # Helper for build subsets
    build_helpers/         # Build helper utilities
    explain.py             # packastack explain
    refresh.py             # packastack refresh ubuntu-archive
    search.py              # packastack search
    clean.py               # packastack clean
    completion.py          # Shell completion handler

  data/
    upstreams.yaml         # Upstream registry (defaults + project overrides)
```

---

## 3. Entry Points & CLI

### Startup Path

```text
uv run packastack → cli.py:app() → Typer dispatches to registered command functions
```

When installed, the `packastack` entry point is on `PATH` and can be invoked directly. During development, always prefix with `uv run` to use the project's virtual environment:

```bash
uv run packastack init
uv run packastack plan nova
uv run python -m packastack     # Equivalent via __main__.py
```

### Exit Codes

Defined in `build/errors.py` (and mirrored in command files):

| Code | Constant | Meaning |
| --- | --- | --- |
| 0 | `EXIT_SUCCESS` | Success |
| 1 | `EXIT_CONFIG_ERROR` | Config or target resolution error |
| 2 | `EXIT_TOOL_MISSING` | Required tool not found |
| 3 | `EXIT_FETCH_FAILED` | Upstream fetch failed |
| 4 | `EXIT_PATCH_FAILED` | gbp patch-queue failed |
| 5 | `EXIT_MISSING_PACKAGES` | Required packages unavailable |
| 6 | `EXIT_CYCLE_DETECTED` | Dependency cycle |
| 7 | `EXIT_BUILD_FAILED` | Build command failed |
| 8 | `EXIT_POLICY_BLOCKED` | Blocked by policy (retired, not latest) |
| 9 | `EXIT_REGISTRY_ERROR` | Upstream registry resolution failed |
| 10 | `EXIT_RETIRED_PROJECT` | Upstream project is retired |
| 11 | `EXIT_DISCOVERY_FAILED` | Package discovery failed |
| 12 | `EXIT_GRAPH_ERROR` | Dependency graph construction failed |
| 13 | `EXIT_ALL_BUILD_FAILED` | One or more builds in build-all failed |
| 14 | `EXIT_RESUME_ERROR` | Build-all resume state corrupted |

---

## 4. Commands Reference

### 4.1 `init`

```bash
packastack init [--prime]
```

**Purpose:** Initialize Packastack — creates config file (`~/.config/packastack/config.yaml`) and all cache directories, runs a tool pre-flight check, and optionally primes Ubuntu archive metadata. Metadata writes are idempotent (safe to re-run).

**What it does:**

1. Runs tool pre-flight: checks for `git`, `gbp`, `dch`, `dpkg-source`, `sbuild`, `schroot`, `gnupg` and reports missing tools with install hints
2. Creates cache directories under `~/.cache/packastack/`
3. Clones `openstack/releases` and `openstack/project-config` repos from OpenDev
4. Creates Ubuntu archive cache structure with `README.txt` + `config.json`
5. Writes default config to `~/.config/packastack/config.yaml` (idempotent)
6. With `--prime`: runs a partial `refresh ubuntu-archive` to warm caches

**Key directories created:**

- `~/.cache/packastack/openstack-releases/` — cloned git repo from OpenDev
- `~/.cache/packastack/openstack-project-config/` — cloned git repo from OpenDev
- `~/.cache/packastack/ubuntu-archive/indexes/` — Packages.gz files
- `~/.cache/packastack/ubuntu-archive/snapshots/` — snapshot metadata
- `~/.cache/packastack/apt-repo/` — local APT repository
- `~/.cache/packastack/upstream-tarballs/` — cached tarballs
- `~/.cache/packastack/build/` — build workspaces

### 4.2 `plan`

```bash
packastack plan PACKAGE [--target TARGET] [--ubuntu-series SERIES]
                        [--force] [--offline] [--output FORMAT]
                        [--include-retired] [--skip-local]
                        [--type {release,snapshot,auto}]
                        [--no-build-deps]

```

**Purpose:** Analyze packaging metadata, resolve dependencies, detect cycles and missing packages, determine build order.

**What it does:**

1. Fetches Ubuntu packaging repositories from Launchpad
2. Parses `debian/control` for each package → dependency graph
3. Selects build types (release vs. snapshot) using `type_selection.py`
4. Detects dependency cycles → suggests edge exclusions
5. Evaluates dependency satisfaction against Ubuntu + cloud archive
6. Generates build order with wave-based parallelism
7. Renders output in DOT, ASCII, HTML, or JSON format

**Key outputs:** Build order wave listing, dependency graph visualization, MIR warnings, cloud archive dependency report.

### 4.3 `build`

```bash
packastack build PACKAGE [--target TARGET] [--ubuntu-series SERIES]
                         [--force] [--offline] [--type {release,snapshot,auto}]
                         [--no-binary] [--source-only]
                         [--ppa-upload] [--ppa TARGET] [--ppa-changes PATH]
                         [--no-ai] [--no-clean]
```

**Purpose:** Build a single OpenStack package from upstream source.

**Build phases (in order):**

1. **Setup** — Create workspace, symlink packaging repo
2. **Plan Validation** — Validate plan against upstream releases
3. **Registry Resolution** — Resolve upstream registry config
4. **Retirement Check** — Check if upstream project is retired
5. **Policy Check** — Validate build policy (force flags, retired projects)
6. **Type Resolution** — Determine RELEASE vs SNAPSHOT
7. **Tarball Acquisition** — Fetch upstream tarball (uscan → official → PyPI → GitHub → git archive)
8. **gbp import-orig** — Import tarball into packaging repo
9. **Patch Queue** — `gbp pq import`, apply patches
10. **Source Build** — `gbp buildpackage -S` or dpkg-buildpackage
11. **Binary Build** — sbuild or dpkg-buildpackage
12. **Artifact Collection** — Collect .deb, .changes, .buildinfo, logs
13. **Local Repo Publish** — Publish to local APT repo
14. **PPA Upload** — Upload to Launchpad PPA if requested
15. **AI Diagnosis** — If build fails, analyze with AI

**Key flags:**

- `--source-only` / `--no-binary`: Build source only, skip binary
- `--ppa-upload`: Upload .changes to PPA after build
- `--no-ai`: Disable AI-powered failure diagnosis
- `--type {release,snapshot,auto}`: Override build type

### 4.4 `build-all`

Implemented within `build.py` when the `--all` flag is passed. Orchestrates building many packages in dependency order.

**What it does:**

1. Discovers all packages in packaging repos
2. Builds dependency graph
3. Filters retired packages
4. Computes parallel build waves
5. Runs builds sequentially or in parallel batches (`--parallel N`, `--keep-going`)
6. Saves/loads state for resume support
7. Generates comprehensive build reports

**Resume support:** Failed packages can be skipped on resume; state is persisted to `build-all-state.json`.

### 4.5 `build` subset builds

Three special package names build an entire class of managed packages in one command, using the `build_subset.py` helper and the same infrastructure as `build --all` (including `--parallel`, `--keep-going`, and `--dry-run`):

```bash
packastack build libraries [--target ...]   # Oslo/shared libraries + Python client libraries
packastack build clients    [--target ...]   # only the Python client libraries
packastack build services   [--target ...]   # only the core services (nova, glance, neutron, ...)
```

Each discovers all packages, classifies them from the OpenStack releases metadata (`SubsetType` in `commands/build_subset.py`), and builds the matching set in dependency order. `--dry-run` lists what would be built without building it. A typical release workflow builds `libraries` first so the local APT repo can satisfy service build-dependencies, then `services`.

### 4.6 `build-rc`

```bash
packastack build rc [--dry-run] [--ppa-upload] [--no-ai]
# also: build rc1, build rc2, ...
```

Discovers all OpenStack packages with RC (release candidate) releases for the current development series and builds them using the build-all infrastructure. Primarily used during OpenStack release preparation. The `rc`, `rc1`, `rc2`, ... aliases are accepted.

### 4.7 `refresh`

```bash
packastack refresh ubuntu-archive [--series SERIES] [--pockets POCKETS]
                                   [--components COMPONENTS]
                                   [--arch ARCHES] [--force] [--offline]

```

Refreshes cached Ubuntu archive Packages.gz indexes. Uses HTTP conditional requests (ETag/If-None-Match and If-Modified-Since) to minimize bandwidth.

### 4.8 `clean`

```bash
packastack clean [--all] [--tarballs] [--workspaces] [--apt-repo]
                 [--expired] [--dry-run] [--force] [--max-age DAYS]

```

Removes cached data: tarballs, build workspaces, APT repo, or expired cache entries.

### 4.9 `completion`

```bash
packastack completion [bash|zsh|fish]

```

Generates shell completion scripts via Typer.

> **Note:** The `explain` and `search` commands that existed in earlier revisions have been removed. Type-selection rationale and dependency-satisfaction reports are still produced by `packastack plan`; target-expression parsing still lives in `packastack.target.resolution` for programmatic use.

---

## 5. Architecture Layers

### 5.1 Core Layer (`packastack.core`)

The foundation of the application.

#### `config.py`

- `load_config()` — Reads `~/.config/packastack/config.yaml`, merges with defaults
- `write_config(data)` — Persists config to disk
- `ensure_config_exists()` — Creates config with defaults on first run
- `get_config_path()` — Returns `Path.home() / ".config" / "packastack" / "config.yaml"`

#### `context.py` — Immutable Config + Mutable Context

**Immutable (frozen dataclasses):**

- `PlanRequest` — package, target, ubuntu_series, force, offline, build_type, build_deps
- `BuildRequest` — Full CLI inputs for build (target, policies, build options, resume, PPA)
- `BuildAllRequest` — CLI inputs for build-all before resolution
- `TargetConfig` — Target series, Ubuntu series, is_development, pocket, component
- `PolicyConfig` — force, offline, include_retired, yes, skip_local, no_ai, no_clean
- `BuildOptions` — build_type, source_only, binary_enabled, builder, arch
- `ResumeConfig` — resume, resume_file, skip_failed

**Mutable:**

- `BuildContext` — Accumulates state during a single build (upstream source, tarball path, patch results, provenance)
- `BuildAllContext` — Accumulates state during build-all (deps graph, wave assignments, per-package results)

#### `run.py` — RunContext

Context manager that creates a per-command run directory under `~/.cache/packastack/build/{package}/{build_id}/` (or `.runs/` for non-package commands). Captures stdout/stderr/events to log files. Supports:

- `log_event()` — Structured JSON log entry
- `add_log_mirror()` — Mirror logs to another directory (for build-all)
- `_relocate()` — Move run directory mid-execution

#### `paths.py`

- `resolve_paths(cfg)` — Resolves `~` and relative paths from config
- `ensure_directories()` — Creates all required cache directories

#### `exceptions.py`

Hierarchical exception types with exit codes:

- `PackastackError` (base, exit 1)
- `ConfigError` (1)
- `PartialRefreshError` (2)
- `OfflineMissingError` (3)
- `CorruptCacheError` (4)
- `MissingPackageError` (5) — includes `missing_packages` dict
- `CycleDetectedError` (6) — includes `cycles` list

### 5.2 Target Layer (`packastack.target`)

Handles Ubuntu series resolution and target expression parsing.

#### `series.py`

`resolve_series(name)` — Resolves "devel" to the current Ubuntu development series via `distro-info --devel`, falling back to a hardcoded default.

#### `arch.py`

- `get_host_arch()` — Maps `platform.machine()` to Debian arch names (x86_64 → amd64, etc.)
- `resolve_arches(arches)` — Replaces "host" with detected architecture

#### `distro_info.py`

Parses `/usr/share/distro-info/ubuntu.csv`. Provides:

- `UbuntuRelease` dataclass with version, codename, LTS status, dates
- `get_current_lts()` — Finds the latest LTS release
- `get_ubuntu_series()` — Get release info for a codename

#### `resolution.py`

Target expression parsing with modes:

- `MatchMode`: EXACT, PREFIX, CONTAINS, GLOB
- `Scope`: SOURCE (Debian source pkg), CANONICAL (upstream project), UPSTREAM (git org/project), DELIVERABLE, REPO
- `TargetExpr` — Parsed expression
- `TargetResolver` — Resolves expressions against upstream registry

### 5.3 Upstream Layer (`packastack.upstream`)

Manages upstream source discovery, acquisition, and verification.

#### `registry.py` — UpstreamsRegistry

The central configuration system for upstream sources.

**Key types:**

- `ReleaseSourceType`: OPENSTACK_RELEASES, GIT_TAGS, PYPI, PINNED
- `TarballMethod`: OFFICIAL, GITHUB_RELEASE, PYPI, GIT_ARCHIVE
- `SignatureMode`: AUTO, REQUIRED_DETACHED, GIT_TAG, GIT_COMMIT, NONE
- `ResolutionSource`: REGISTRY_EXPLICIT, REGISTRY_DEFAULTS, LEGACY_OPENSTACK_RELEASES

**How it works:**

1. Loads `upstreams.yaml` (packaged data file)
2. For each project: checks for explicit entry → applies defaults
3. Most OpenStack projects use defaults (OpenDev git, openstack/releases governance, official tarballs)
4. Non-standard projects (gnocchi, alembic, rally, networking-l2gw) have explicit entries
5. Falls back to legacy openstack/releases data for projects not in registry

**Key methods:**

- `resolve(project_name)` → `ResolvedUpstream`
- `get_canonical(project_name)` → canonical identifier

#### `source.py`

- `UpstreamSource` — Captures version, git_ref, tarball_url, signature_url, build_type
- `TarballResult` — Download + verification result
- `SnapshotAcquisitionResult` — Git snapshot result with sha, branch, tarball_path
- `download_and_verify_tarball()` — Download with SHA256 + signature verification
- `generate_snapshot_tarball()` — Create tarball from git repo at a ref

#### `releases.py`

Parses the `openstack/releases` repository's YAML data:

- `ReleaseVersion` — Single version with beta/RC/final detection
- `ProjectRelease` — Per-project release history, model, type
- `load_openstack_packages()` — Load package → upstream project mapping
- `load_project_releases()` — Load release data for a specific project
- `load_series_info()` — Load series metadata
- `get_current_development_series()` — Find active dev series
- `is_snapshot_eligible()` — Determines if a project can use snapshot builds

#### `retirement.py`

Two-source retirement detection:

1. **Authoritative:** `openstack/project-config`'s `gerrit/projects.yaml` — projects with "RETIRED" in description
2. **Inference:** `openstack/releases` data — not seen in >= 3 cycles → possibly_retired

Types: `RetirementStatus` (ACTIVE, RETIRED, POSSIBLY_RETIRED, UNKNOWN), `RetirementInfo`, `RetirementChecker`

#### `gitfetch.py` — GitFetcher

Clones/updates packaging repos from `https://git.launchpad.net/~ubuntu-openstack-dev/ubuntu/+source/{package}`. Uses file-based locking (`fcntl`) to prevent concurrent clones of the same package. Configurable lock timeout (default 5 minutes).

#### `tarball_cache.py`

Tarball caching with TTL. Manages `~/.cache/packastack/upstream-tarballs/`. Provides cache entry lookup, expiration, size tracking, and cleanup.

### 5.4 Planning Layer (`packastack.planning`)

The "brain" of Packastack — determines what to build and in what order.

#### `graph.py` — DependencyGraph

Directed acyclic graph where nodes are source packages and edges represent build dependencies.

- `topological_sort()` — Build order
- `compute_waves()` — Parallel build wave assignment (packages with no inter-dependencies can build in parallel)
- `detect_cycles()` — DFS-based cycle detection
- `compute_forced_by()` — Critical dependency chains

#### `graph_builder.py`

Builds the `DependencyGraph` from `debian/control` files:

- Parses Build-Depends and Build-Depends-Indep
- Resolves binary package names → source package names via PackageIndex
- Handles alternatives (`|`), architecture qualifiers, version constraints
- `OPTIONAL_BUILD_DEPS` — Dependencies that are optional for the build

#### `type_selection.py`

The most complex planning module (1385 lines). Determines whether each package should be built as RELEASE or SNAPSHOT.

**Key types:**

- `BuildType`: RELEASE, SNAPSHOT
- `CycleStage`: PRE_FINAL, POST_FINAL, UNKNOWN
- `DeliverableKind`: SERVICE, LIBRARY, CLIENT_LIBRARY, CLIENT, HORIZON_PLUGIN, TEMPEST_PLUGIN, OTHER

**Selection logic:**

1. Check if upstream has a tagged release for the target series
2. For PRE_FINAL series: libraries prefer snapshots, services prefer releases (if available)
3. For POST_FINAL: prefer releases, fall back to snapshots
4. Respect `cycle-with-rc`, `cycle-with-intermediary`, `cycle-trailing` release models
5. Handle independent projects (not governed by openstack/releases)

#### `package_discovery.py`

Discovers packages in packaging repositories, maps source package names to upstream projects.

#### `dependency_satisfaction.py`

Evaluates whether build dependencies can be satisfied from:

- Ubuntu development series
- Current LTS
- Cloud Archive
- Local APT repo (previously built packages)

#### `cycle_suggestions.py`

When dependency cycles are detected, suggests which edges to exclude to break cycles. Considers edge criticality and impact on build order.

#### `build_all_state.py`

State machine for build-all operations. Tracks per-package status (PENDING, BUILDING, SUCCESS, FAILED, SKIPPED, BLOCKED), failure types, and timestamps. Supports serialization for resume.

### 5.5 Build Layer (`packastack.build`)

The execution engine that actually builds packages.

#### `types.py` — Build Data Types

- `ResolvedTargets` — Target series, Ubuntu series, pocket, component
- `WorkspacePaths` — Build workspace, packaging repo, tarball dest, etc.
- `BuildInputs` — Consolidated inputs for a build operation
- `PhaseResult` — Success/failure with exit code, message, data
- `BuildOutcome` — Final build result with phase results, provenance, artifacts

#### `errors.py` — Error Handling

- `phase_error(phase, message, run, exit_code)` — Logs error, writes summary, returns exit code
- `phase_warning(phase, message, run)` — Logs warning without failing
- `log_phase_event()` — Structured event logging

#### `mode.py` — BuildMode

Controls build configuration:

- `Builder`: SBUILD (clean chroot) or DPKG (dpkg-buildpackage)
- `BuildMode.sources` + `BuildMode.binaries` — What to produce
- Factory methods: `source_only()`, `full_build()`

#### `provenance.py` — BuildProvenance

Records detailed provenance for auditability:

- `UpstreamProvenance` — URL, ref, sha, branch
- `ReleaseSourceProvenance` — Type, deliverable, tag_regex, resolved version
- `TarballProvenance` — Method, URL, path, sha256
- `BuildEnvironment` — Host, arch, distribution, tools used

#### `phases.py` — Build Phases

Reusable phase functions used by both single build and build-all:

- `check_retirement_status()` → `RetirementCheckResult`
- `resolve_upstream_registry()` → `RegistryResolutionResult`
- `check_policy()` → `PolicyCheckResult`
- `load_package_indexes()` → `PackageIndexes`
- `check_tools()` → `ToolCheckResult`
- `ensure_schroot_ready()` → `SchrootSetupResult`

#### `single_build.py` — Single Package Build

The complete single-package build pipeline. Each phase is a function taking a `BuildContext` and returning a `PhaseResult`. Phases are independent and composable.

#### `all_runner.py` — Build-All Orchestration

Orchestrates building many packages:

- `_run_build_all()` — Main entry point
- `_run_sequential_builds()` — One at a time in dependency order
- `_run_parallel_builds()` — Wave-based parallel execution
- Supports resume from saved state

#### `tarball.py` — Tarball Acquisition

Multi-strategy tarball acquisition:

1. **uscan** (primary) — Uses debian/watch to download + verify
2. **Official URL** — Direct download from tarballs.opendev.org
3. **PyPI** — Download from pypi.org
4. **GitHub Releases** — Download from GitHub
5. **Git archive** — Generate from git repository

#### `git_helpers.py` — Git Operations

- `git_commit()` — Standardized commit with staging, message, optional GPG
- `extract_upstream_version()` — Parse version from changelog/git tags
- `ensure_no_merge_paths()` — .gitattributes management
- `maybe_enable_sphinxdoc()` — debian/rules sphinxdoc addons
- `GitCommitError` — Typed commit failures

#### `sbuild.py` — Sbuild Integration

Clean-room binary builds using sbuild:

- Auto-discovers sbuild config (user and global)
- Binds local APT repo into chroot at `/srv/packastack-apt`
- Captures stdout/stderr to log files
- Collects artifacts (.deb, .changes, .buildinfo)

#### `sbuildrc.py` — Sbuild Config Discovery

Finds sbuild configuration directories by parsing `~/.sbuildrc` and `/etc/sbuild/sbuild.conf`.

#### `schroot.py` — Schroot Management

Creates and manages schroot sessions for sbuild. Handles repo mounting/unmounting.

#### `collector.py` — Artifact Collection

Discovers and collects sbuild output artifacts and log files from multiple candidate directories. Computes SHA256 hashes, copies files, generates artifact reports.

### 5.6 Debpkg Layer (`packastack.debpkg`)

Debian packaging file parsing and manipulation.

#### `control.py`

`ParsedDependency` — Represents a single Debian dependency with:

- name, relation (>=, <=, =, >>, <<), version
- arch_qualifiers (e.g., `[amd64 !i386]`)
- alternatives (pipe-separated options)

`parse_control()` — Parses `debian/control` into sections with Build-Depends, Depends, etc.

#### `changelog.py`

Parses `debian/changelog` using python-debian. Extracts version, distribution, urgency, changes entries.

#### `version.py`

`ParsedVersion` — Full Debian version with epoch, upstream_version, debian_revision. Supports comparison using python-debian's `Version` class with string fallback.

`extract_upstream_version()` — Strips epoch and debian revision to get the upstream component.

#### `gbp.py`

Git-buildpackage wrapper:

- `PatchHealthReport` — Per-patch health (success, failure_reason, suggested_action)
- `PQResult` — Patch queue operation result
- `BuildResult` — Build operation result with dsc_file, changes_file
- `run_command()` — Execute gbp commands with output capture
- `import_orig()` — Import upstream tarball
- `pq_import()` / `pq_export()` — Patch queue operations
- `build_source()` / `build_binary()` — Build commands

#### `watch.py`

`debian/watch` file parser and uscan integration:

- `DetectedWatchMode` — OPENSTACK_TARBALL, PYPI, GITHUB_RELEASE, GITHUB_TAGS, etc.
- `WatchParseResult` — Parsed watch file with source type and comparison
- `UscanResult` — uscan execution result
- `detect_upstream_source_type()` — Heuristic watch file analysis
- `check_watch_registry_mismatch()` — Compare watch vs. registry

#### `rules.py`

Utilities for patching `debian/rules`:

- `add_doctree_cleanup()` — Fix lintian `package-contains-python-doctree-file` warning

#### `gbpconf.py`

`gbp.conf` management — parsing, generating, and comparing gbp configuration.

#### `launchpad_yaml.py`

Parses `launchpad.yaml` packaging metadata files.

### 5.7 APT Layer (`packastack.apt`)

Ubuntu archive interaction and local APT repository management.

#### `archive.py` — ArchiveFetcher

Fetches Packages.gz files from Ubuntu mirrors with HTTP conditional requests:

- `build_url(mirror, series, pocket, component, arch)` — Construct URL
- `fetch_packages_gz(url, cache_dir)` — Download with ETag/If-Modified-Since
- Returns `FetchResult` with etag, sha256, size, was_cached flag

#### `localrepo.py`

Manages the local APT repository at `~/.cache/packastack/apt-repo/`:

- `publish_package(changes_file)` — Add .deb to local repo
- `regenerate_indexes()` — Rebuild Packages, Packages.gz, Release
- `DebPackageInfo` — Extracted .deb metadata
- Used so that packages built earlier can satisfy Build-Depends for later packages

#### `packages.py`

`PackageIndex` — Merged index of:

- Ubuntu dev series packages
- Cloud Archive packages
- Previously built local packages
- Provides `find_package()`, `find_source()`, `find_packages_providing()`

### 5.8 Logs Layer (`packastack.logs`)

Report generation and visualization.

#### `plan_graph.py`

`PlanGraph` — Build dependency graph visualization:

- `render_dot()` — Graphviz DOT format
- `render_ascii()` — ASCII tree listing
- `render_html()` — Self-contained HTML with interactive graph
- `render_waves()` — Wave-based parallel build order
- `render_build_order_list()` — Sequential build order listing
- `render_json()` — Machine-readable JSON
- `write_plan_graph_reports()` — Write all formats to disk

#### `type_selection.py`

Type selection reporting:

- `render_compact_summary()` — One-line per package
- `render_console_table()` — Rich table format
- `write_type_selection_reports()` — Write reports to disk

#### `watch_resolution.py`

Watch file resolution reports — documents how upstream versions were discovered.

#### `plan_reports.py`

Plan command report helpers — type-selection rationale, dependency-satisfaction details, cloud-archive requirements, and MIR warnings produced by `packastack plan`.

#### `all_reports.py`

Build-all summary reports — aggregate build results, timing, failure analysis.

### 5.9 AI Layer (`packastack.ai`) — build-doctor skill system

AI-powered build failure diagnosis (optional, requires API key). The system is a deterministic Python dispatcher plus pluggable **skills** on disk. Skills are folders containing a `SKILL.md` file with YAML frontmatter (name, description, output contract, trigger regexes) and a prompt body. Adding a new failure-class handler is a folder drop — no Python changes.

#### `client.py`

Thin wrapper around OpenAI-compatible APIs:

- `get_api_key()` — Env vars (`PACKASTACK_AI_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` backward compat) or config
- `call_ai(prompt, model, temperature)` → `AIResponse`
- Default: OpenRouter (<https://openrouter.ai/api/v1>)
- `is_ai_available()` — Check if API key is configured

#### `skills.py` — Skill loader

Loads `src/packastack/skills/<name>/SKILL.md` (markdown + YAML frontmatter, no Python). Supports a `PACKASTACK_SKILLS_DIR` override for dev copies. Skills declare `name`, `description`, `output_contract`, optional `triggers.log_patterns`, and `requires_context` (collector names).

#### `runner.py` — Skill orchestration

`run_skill()` is the single entry point for invoking an AI skill. It loads the skill, expands `{{skills_menu}}` placeholders in the system prompt, assembles the user message from the collectors named in `requires_context`, calls the AI, and parses the response according to the skill's `output_contract`. All errors are caught and returned as `SkillResult(success=False, ...)` so callers don't need try/except.

#### `collectors.py` — Context collectors

A *collector* is a named callable returning a formatted string section for the user message (e.g. the sbuild failure log excerpt, `debian/control`, `debian/rules`). Skills reference collectors by name via `requires_context`.

#### `contracts.py` — Output contract parsers

Every skill declares an `output_contract` in its frontmatter. Parsers here convert the raw AI response into a typed payload (patch + DEP3 headers, diagnosis+action, guidance, etc.).

#### `triggers.py` — Trigger-based routing

Cheap regex routing: when a failure log matches a skill's `triggers.log_patterns`, the dispatcher routes directly to that skill without a router AI call.

#### `build_diagnosis.py`

`diagnose_build_failure()` — entry point used by the build pipeline on sbuild failure:

1. Extracts the failure section from the sbuild log.
2. Tries `triggers.py` regex pass over every skill. A match routes directly.
3. Otherwise asks the `build-doctor` router skill to pick a specialist (returns a structured dispatch with confidence, evidence, fallbacks).
4. Validates the pick against a confidence threshold (default 0.5) and the installed skill registry; walks the fallback chain; drops to generic `build-patch` as a last resort.
5. Runs the chosen specialist via `runner.run_skill()`, validates any patch with `git apply --check` (one correction retry via `patch-correction`), and commits the result.

#### `patch_diagnosis.py`

`diagnose_patch_failure()` — handles `gbp pq import` failures:

1. `attempt_mechanical_refresh()` — Try offset/fuzz fixes.
2. `auto_drop_upstreamed_patches()` — Detect patches already applied upstream via `git apply --check --reverse`, then drop the file, update `debian/patches/series`, and commit.
3. `refresh_failing_patch()` — AI-generated patch refresh routed to `patch-refresh`.

#### `prompts.py`

Shared prompt helpers and `{{skills_menu}}` expansion for skill system prompts.

#### `memory.py`

Persistent AI memory — saves diagnosis results and attempted patches to `ai-memory.json` in the run directory so subsequent builds can tell skills what was already tried.

#### Skills shipped (`src/packastack/skills/`)

| Skill | Output | Role |
|---|---|---|
| `build-doctor` | dispatch | Router — picks the specialist |
| `build-patch` | patch | Generic patch author; safe fallback |
| `patch-diagnosis` | diagnosis | Decides whether an existing quilt patch can be dropped (already upstream?) |
| `patch-refresh` | patch | Refreshes fuzzy patches after upstream drift |
| `patch-correction` | patch | Fixes a patch that failed `git apply --check` |
| `python-compat` | patch | Interpreter bumps (`distutils`/`imp`/`cgi` removed, `ast.Str` → `Constant`, etc.) |
| `library-sync-advisor` | guidance | Detects stale oslo / python-\*client deps and advises a library sync instead of a patch |

---

## 6. Data Model

### 6.1 Upstreams Registry (`upstreams.yaml`)

Packaged at `src/packastack/data/upstreams.yaml`. A versioned (v2) registry defining upstream source configuration.

**Schema:**

```yaml
version: 2
defaults:
  upstream:
    type: git              # git, github
    host: opendev           # opendev, github
    default_branch: master
  release_source:
    type: openstack_releases
    strict: true            # Must have releases entry
  tarball:
    prefer: [official]
  signatures:
    mode: auto
  requirements:
    files: [pyproject.toml, requirements.txt, ...]

projects:
  <project_name>:
    canonical: <org/project>
    common_names: [...]
    retired: true|false
    upstream: { ... }       # Override defaults
    release_source: { ... } # Override defaults
    tarball: { ... }        # Override defaults
    signatures: { ... }     # Override defaults
    watch:
      expect: { mode, base_url, url, package_name }

```

**Resolution logic:**

1. Defaults apply to all projects NOT in the `projects` map
2. Projects in `projects` map merge with defaults (explicit keys override)
3. Projects not in either: use legacy `openstack/releases` fallback

### 6.2 Build Type Selection

The `type_selection.py` module determines `BuildType.RELEASE` vs `BuildType.SNAPSHOT`:

1. Determine **CycleStage** (PRE_FINAL vs POST_FINAL)
   - PRE_FINAL: OpenStack series hasn't had final release yet
   - POST_FINAL: Series has been released
2. Classify **DeliverableKind**
   - From releases data type field (SERVICE, LIBRARY, CLIENT_LIBRARY, ...)
   - Fallback to heuristic (name patterns)
   - Default: OTHER
3. Check **release availability**
   - Does project have a tagged release for target series?
   - Is it beta, RC, or final?
4. Apply **selection rules**:
   - PRE_FINAL + LIBRARY → SNAPSHOT (libraries change rapidly during dev)
   - PRE_FINAL + SERVICE → RELEASE if available, else SNAPSHOT
   - POST_FINAL + has releases → RELEASE
   - POST_FINAL + no releases → SNAPSHOT
   - Force flags override
   - Consider release model (cycle-with-rc favors releases)

### 6.3 Dependency Graph

```text
DependencyGraph
├── nodes: {name → GraphNode}
│   ├── name: str
│   ├── version: str
│   ├── needs_rebuild: bool
│   ├── rebuild_reason: str
│   └── mir_warnings: list[str]
├── edges: {from → {to, ...}}        # A depends on B
└── reverse_edges: {to → {from, ...}}  # B is needed by A

Operations:
├── topological_sort() → list[str]
├── compute_waves() → list[list[str]]   # Parallel groups
├── detect_cycles() → list[list[str]]
├── compute_forced_by() → dict[str, list[str]]
└── get_transitive_deps() → set[str]

```

### 6.4 Build-All State Machine

```text
BuildAllState
├── build_id: str
├── target: str
├── packages: {name → PackageState}
│   ├── status: PENDING | BUILDING | SUCCESS | FAILED | SKIPPED | BLOCKED
│   ├── failure_type: FailureType | None
│   ├── start_time / end_time: datetime | None
│   ├── build_id: str
│   └── error: str
├── waves: list[list[str]]
├── completed_wave: int
└── failed_packages: list[str]

```

State transitions:

```text
PENDING → BUILDING → SUCCESS
PENDING → BUILDING → FAILED
PENDING → BLOCKED (dependency failed)
PENDING → SKIPPED (previously failed, resume with --skip-failed)

```

---

## 7. Key Workflows

### 7.1 Single Package Build Flow

1. CLI: `build(package, target, ...)`
2. Create `BuildRequest` from CLI args
3. `With RunContext("build", package=package):`
4. Resolve targets (series, arch, pocket, component)
5. Clone/update packaging repo from Launchpad
6. Plan validation: Verify package exists, upstream has source
7. Registry resolution: Load `upstreams.yaml` → `ResolvedUpstream`
8. Retirement check: Is upstream project retired?
9. Policy check: Force flags, retired handling
10. Type resolution: RELEASE vs SNAPSHOT
11. Tarball acquisition:
    1. uscan (`debian/watch`)
    2. Official URL (`tarballs.opendev.org`)
    3. PyPI
    4. GitHub Releases
    5. git archive (generate from repo)
12. `gbp import-orig`: Import tarball into packaging repo
13. `gbp pq import`: Apply patches from `debian/patches`
14. Source build: `gbp buildpackage -S` or `dpkg-buildpackage`
15. Binary build (optional): `sbuild` or `dpkg-buildpackage`
16. Artifact collection: Copy `.deb`, `.changes`, `.buildinfo`, logs
17. Local repo publish: Add to local APT repo
18. PPA upload (optional): `dput` to Launchpad
19. AI diagnosis (on failure): Analyze logs, suggest fixes

### 7.2 Build-All Orchestration Flow

1. CLI: `build --all [--target ...]`
2. Initialize/load state (for resume)
3. Discover packages in packaging repos
4. Build dependency graph from `debian/control`
5. Filter retired packages
6. Compute waves (parallel build groups)
7. For each wave:
   1. Build packages in parallel (`ThreadPoolExecutor`)
   2. After each successful build: publish to local APT repo
   3. Failed packages: record failure, continue (or stop if critical)
8. Save state (for resume)
9. Generate build-all reports
10. Exit: 0 if all succeeded, `EXIT_ALL_BUILD_FAILED` otherwise

### 7.3 Plan Command Flow

1. CLI: `plan(package, target, ...)`
2. `With RunContext("plan"):`
3. Fetch packaging repos from Launchpad
4. Parse `debian/control` → dependency graph
5. Select build types (RELEASE vs SNAPSHOT)
6. Check dependency cycles → suggest exclusions
7. Evaluate dependency satisfaction:
   - Ubuntu dev series
   - Current LTS
   - Cloud Archive
8. Compute build order + waves
9. Generate reports:
   - Build order (console)
   - Dependency graph (DOT/ASCII/HTML/JSON)
   - Type selection summary
   - MIR warnings
   - Cloud archive deps

---

## 8. Configuration

Configuration file: `~/.config/packastack/config.yaml`

Generated on first `packastack init` with sensible defaults:

```yaml
paths:
  cache_root: ~/.cache/packastack
  openstack_releases_repo: ~/.cache/packastack/openstack-releases
  openstack_project_config: ~/.cache/packastack/openstack-project-config
  ubuntu_archive_cache: ~/.cache/packastack/ubuntu-archive
  local_apt_repo: ~/.cache/packastack/apt-repo
  upstream_tarballs: ~/.cache/packastack/upstream-tarballs
  build_root: ~/.cache/packastack/build

defaults:
  upstream_target: devel
  ubuntu_series: devel
  ubuntu_pockets: [release, updates, security]
  ubuntu_components: [main, universe]
  ubuntu_arches: [host, all]
  refresh_ttl: 6h
  mir_policy: warn
  cloud_archive: null
  upload_ppa: null

mirrors:
  ubuntu_archive: http://archive.ubuntu.com/ubuntu
  ubuntu_openstack_git: https://git.launchpad.net/~ubuntu-openstack-dev/ubuntu/+source

repo_name_overrides:
  tap-as-a-service: neutron-taas
  trove: openstack-trove

git:
  launchpad_username: null

launchpad_bugs:
  client-lib-release: null
  milestone2: null
  service-rc1: null
  service-release: null

behavior:
  offline: false
  snapshot_archive_on_build: true

ai:
  api_key: null       # Fallback; env vars take precedence
  base_url: https://openrouter.ai/api/v1
  model: anthropic/claude-sonnet-4-5-20250929
  max_tokens: 8192
  timeout: 120
```

### Environment Variables

| Variable | Effect |
| --- | --- |
| `PACKASTACK_AI_API_KEY` | AI API key (highest priority) |
| `OPENAI_API_KEY` | AI API key (fallback) |
| `ANTHROPIC_API_KEY` | AI API key (backward compat) |
| `DEBEMAIL` | Git commit author email |
| `DEBFULLNAME` | Git commit author name |

---

## 9. Testing

### Structure

```text
tests/
  conftest.py           # Shared fixtures: temp_home, mock_config, mock_cache_dirs,
                        #   sample_packages_gz, mock_responses, mock_registry,
                        #   make_build_request, make_resolved_target
  core/                 # Core module tests
  commands/             # Command tests (largest: test_build.py at 134KB)
  planning/             # Planning module tests
  debpkg/               # Debpkg module tests
  apt/                  # APT module tests
  ai/                   # AI module tests
  logs/                 # Logs module tests
  upstream/             # Upstream module tests

```

### Coverage

- Target: 95% (enforced by `--cov-fail-under=95`)
- Branch coverage enabled (`branch = true`)
- Many integration-heavy modules excluded from coverage (listed in `tool.coverage.run.omit`)

### Running Tests

```bash
tox -e unit                           # All tests with coverage (preferred)
uv run pytest                          # Direct: all tests with coverage
uv run pytest tests/core/              # Specific module
uv run pytest -k "test_name"           # Specific test
uv run pytest --cov-report=html        # HTML coverage report
```

### Linting and Formatting

```bash
tox -e fmt                                   # Auto-fix lint + format (preferred)
tox -e pep8                                  # CI gate: check only, no modifications
uv run ruff check --fix packastack/ tests/  # Direct: auto-fix lint issues
uv run ruff check packastack/ tests/        # Direct: check without modifying
uv run ruff format packastack/ tests/       # Direct: format code
```

### Type Checking

```bash
tox -e mypy                            # Preferred (checks src/ via uv run --frozen --isolated --extra=dev)
uv run mypy src/packastack             # Direct: strict mode, src only
```

> Tests are not yet type-checked by the `tox -e mypy` env (see the TODO in `tox.ini`); enable once test functions carry return-type annotations.

### Tox Automation

All testing, formatting, linting, and type checking can be run via `tox`. The
`tox.ini` at the project root defines the following environments:

| Env | Description | Modifies code? |
| --- | --- | --- |
| `tox -e fmt` | Auto-fix lint issues (`ruff check --fix`) then format (`ruff format`) | Yes |
| `tox -e pep8` | Check-only: show format diff (`ruff format --diff`) and lint (`ruff check`) | No |
| `tox -e mypy` | Type-check `src/` and `tests/` with mypy via `uv run` | No |
| `tox -e unit` | Run `pytest` via `uv run` with dev dependencies | No |
| `tox -e cover` | Run tests with `coverage`, produce HTML/XML/term reports | No |
| `tox -e mypy` | Type-check `src/packastack/` with mypy via `uv run` (tests TODO) | No |
Run all default environments (pytest, fmt, mypy) at once:

```bash
tox
```

Or run a specific environment:

```bash
tox -e fmt      # Auto-format and auto-fix
tox -e pep8     # CI gate — check only, no modifications
tox -e mypy     # Type checking
tox -e unit      # Tests only
tox -e cover     # Tests with full coverage report
```

Pass extra arguments to pytest via `{posargs}`:

```bash
tox -e unit -- tests/core/ -k "test_config"
```

---

## 10. Development Workflow

### Setup

Packastack targets **Python 3.14+** (see `.python-version` and `requires-python` in `pyproject.toml`). The project uses [uv](https://docs.astral.sh/uv/) for dependency management and `tox` for testing, linting, and type checking.

```bash
git clone https://github.com/canonical/packastack
cd packastack
uv sync                              # Install all dependencies into a venv
tox                                  # Run all default envs (pytest, fmt, mypy)
uv run packastack init               # Initialize Packastack
```

All development commands use `uv run` to execute within the project's managed virtual environment. The `uv.lock` file pins all dependency versions for reproducible builds. Use `tox` (or the individual `tox -e <env>` commands) for the canonical test/lint/type-check pipeline.

### Making Changes

1. **Understand the layer** — Changes to build logic touch `build/`, CLI changes touch `commands/`, etc.
2. **Write tests first** — Delegate test authoring to the Tester agent
3. **Implement the change** — Follow existing patterns; use immutable dataclasses for config, mutable contexts for state
4. **Auto-format and lint** — `tox -e fmt`
5. **Type-check** — `tox -e mypy`
6. **Run the full test suite** — `tox -e unit`
7. **Or run everything at once** — `tox`

### Design Conventions

- **Immutable configs:** `@dataclass(frozen=True)` — `TargetConfig`, `PolicyConfig`, `BuildOptions`, `BuildRequest`
- **Mutable context:** `@dataclass` — `BuildContext` (accumulates state)
- **Phase functions:** `def phase_name(ctx, ...) -> PhaseResult | tuple[PhaseResult, Data]`
- **Error handling:** Specific exception types with exit codes; never bare `Exception`
- **CLI:** Typer with `typer.Option()` for mandatory args, `--flag/--no-flag` for booleans
- **Package `__init__.py`:** Only `__all__` + imports; no implementations
- **Entry points:** `cli.py` for Typer app, `__main__.py` for `uv run python -m packastack`

### Key Patterns

1. **RunContext wrapping** — Most command functions wrap their logic in `with RunContext("command_name")`
2. **Activity logging** — `activity(phase, message)` writes to real stdout even when RunContext redirects
3. **Phase-based execution** — Build phases are composable functions returning `PhaseResult`
4. **Lazy imports for heavy modules** — `launchpadlib`, `git` are imported only when needed
5. **File-based locking** — `GitFetcher` uses `fcntl.flock()` for concurrent clone prevention

### Common Pitfalls

- **RunContext required** — Always wrap command logic in `RunContext` for proper log capture
- **Build paths** — Build artifacts go to `~/.cache/packastack/build/{package}/{build_id}/`
- **Offline mode** — Many commands support `--offline`; fails gracefully with `OfflineMissingError` (exit 3)
- **Exit code consistency** — Command files define their own exit code constants matching `build/errors.py`
- **Registry fallback** — New upstream projects not in `upstreams.yaml` fall back to `openstack/releases` data; only add entries for projects that deviate from OpenDev defaults
