# Repository Guidelines

## Project Structure & Module Organization
Gameplay engines, devices, and serialization code live under `src/`. Key folders: `engine/` (record/playback/prebake loops), `input/` & `output/` (hardware adapters), `io/` (JSON/MessagePack serializers), `common/` (timers, logging, shared utilities), `c_api/` (flat C surface for P/Invoke and other consumers), `replayer/` (CLI playback tooling), and `gui_wpf/` (WPF shell, MVVM view models, services, and interop). Runtime defaults live in `config/`, helpers in `scripts/`, external dependencies in `external/`, and NuGet artifacts under `packages/`. Tests span the native C API suite in `tests/c_api_tests.cpp` plus .NET/xUnit projects under `tests/Interop` and `tests/Services`. Keep new modules beside their peers and hook them up from the root `CMakeLists.txt` (and `src/gui_wpf/GameInputRecorder.csproj` when UI code is touched).

## Build, Test, and Development Commands
- `pwsh scripts/build.ps1 -Configuration Debug -RunTests -BuildGui` is the recommended one-step dev loop: it configures CMake into `build/`, builds native libraries and `recorder_c_api.dll`, runs `ctest`, and builds the WPF GUI.
- For manual control, `cmake -S . -B build -G "Visual Studio 17 2022" -A x64 -DCMAKE_BUILD_TYPE=Debug` configures an out-of-source Windows build, then `cmake --build build --config Release --parallel` compiles the native libraries and `recorder_c_api` (with `BUILD_WPF_GUI=ON` it also invokes `dotnet build` for the WPF shell).
- `ctest --test-dir build -C Release --output-on-failure` executes the C API GoogleTests (for example, the `c_api_tests` target wired up from `tests/CMakeLists.txt`; requires `-DBUILD_TESTS=ON`).
- `dotnet test tests/Interop/InteropTests.csproj` and `dotnet test tests/Services/ServicesTests.csproj` run the managed interop and service-layer test suites (ensure `src/gui_wpf` and `recorder_c_api.dll` are built first).
- `pwsh scripts/run_verification_tests.ps1` chains the high-level recording/replay smoke suite; run it before sharing binaries.
- `pip install -r requirements.txt` installs optional Python analyzers used by helper scripts.

## Coding Style & Naming Conventions
Use 4-space indentation, braces on the same line, and keep headers self-contained. Classes and structs use `PascalCase`, public methods `camelCase`, and private data members end in `_`. Favor RAII for HANDLE/QPC lifetimes and emit diagnostics through `LOG_*` macros instead of raw I/O. In WPF or C# interop code, follow the `GameInputRecorder` namespace layout and keep filenames aligned with the contained class or view.

## Testing Guidelines
New C++ behavior in the engine, I/O, or C API should be covered by focused GoogleTests in the native suite (for example, by extending `tests/c_api_tests.cpp` or wiring a new `*_tests.cpp` from `tests/CMakeLists.txt`). Run `ctest` locally (preferably in Release) and attach representative recordings if your change touches timing heuristics or file formats. Managed interop and GUI behavior should be exercised through the xUnit projects in `tests/Interop` and `tests/Services` (run via `dotnet test`); keep tests fast, deterministic, and focused on public C API surfaces where possible. Add fixtures under `tests/fixtures/` when introducing serialization formats so replay regressions stay reproducible.

## Commit & Pull Request Guidelines
Match the repo history: Chinese subject lines that lead with a scope plus a short summary (for example, `回放器: 修复自适应等待漂移` or `代码质量：消除全部MSVC编译警告`). Group one logical change per commit and document reproduction or perf numbers in the body. Pull requests should link issues, describe risk areas (devices, sampling rate, GUI), and note the results of `cmake --build`, `ctest`, `dotnet test`, and `scripts/run_verification_tests.ps1`. Attach screenshots or log snippets whenever GUI flows or timing metrics change.

## Configuration & Security Tips
Check `config/default_settings.json` and `config/logging.json` into source control, but keep OBS keys, proxy creds, and hardware identifiers in user overrides. When sharing recordings, scrub `system_info` blocks and validate MessagePack payloads with `src/common/file_format.h` before replaying on lab machines.

# AGENTS Guidelines

## Windows Environment Notice

- Prefer PowerShell (`pwsh`/`powershell`) when on Windows.
- Prefer using pwsh.exe to run `pnpm <script>` when on WSL2.
- WSL2 may be used for non-package-manager commands only (e.g., `rg`, `tar`). Avoid running Node builds in WSL due to OS mismatch.
- WSL2 cross-drive performance: accessing repos under `/mnt/c|d|e/...` goes through a filesystem bridge and can be slower for full scans. Prefer scoping to subtrees, excluding heavy folders, or running the same searches with native Windows binaries in PowerShell for large/iterative scans.
- Do not auto-run dependency installs. The user must run `pnpm install` in Windows PowerShell manually and then confirm completion.
- Do not modify `package.json`/lockfiles to add or update dependencies without explicit user approval. Propose dependencies in `/spec` or `/plan`, and ask the user to run `pnpm add <pkg>` (or `pnpm install`) and confirm.
- Do not call Unix text tools directly in PowerShell (e.g., `sed`, `awk`, `cut`, `head`, `tail`). Use PowerShell-native equivalents instead:
  - `head` -> `Select-Object -First N`
  - `tail` -> `Get-Content -Tail N`
  - paging -> `Out-Host -Paging` or `more`
  - substitution/replace -> `-replace` with `Get-Content`/`Set-Content`

## Tool Priority

- Filename search: `fd`.
- Text/content search: `rg` (ripgrep).
- AST/structural search: `sg` (ast-grep) - preferred for code-aware queries (imports, call expressions, JSX/TSX nodes).

### AST-grep Usage

- Announce intent and show the exact command before running complex patterns.
- Common queries:
  - Find imports from `node:path` (TypeScript/TSX):
    - `ast-grep -p "import $$ from 'node:path'" src --lang ts,tsx,mts,cts`
  - Find CommonJS requires of `node:path`:
    - `ast-grep -p "require('node:path')" src --lang js,cjs,mjs,ts,tsx`
  - Suggest rewrite (do not auto-apply in code unless approved):
    - Search: `ast-grep -p "import $$ from 'node:path'" src --lang ts,tsx`
    - Proposed replacement: `import $$ from 'pathe'`

### Search Hygiene (fd/rg/sg)

- Exclude bulky folders to keep searches fast and relevant: `.git`, `node_modules`, `coverage`, `out`, `dist`.
- Prefer running searches against a scoped path (e.g., `src`) to implicitly avoid vendor and VCS directories.
- Examples:
  - `rg -n "pattern" -g "!{.git,node_modules,coverage,out,dist}" src`
  - `fd --hidden --exclude .git --exclude node_modules --exclude coverage --exclude out --exclude dist --type f ".tsx?$" src`
- ast-grep typically respects `.gitignore`; target `src` to avoid scanning vendor folders:
  - `ast-grep -p "import $$ from '@shared/$$'" src --lang ts,tsx,mts,cts`
  - If needed, add ignore patterns to your ignore files rather than disabling ignores.

## Temporary Research Files

- Canonical location: store all temporary research artifacts (downloaded READMEs, API docs, scratch notes) under `docs/research/`.
- Do not place temporary files at the repository root or outside `docs/research/`.
- Commit policy: avoid committing temporary files unless they are necessary for traceability during `/spec` or `/plan`. If committed, keep the scope minimal and store them under `docs/` only.
- Naming: use descriptive names with date or task context (e.g., `docs/research/2025-09-11-wsl-notes.md`).
- Cleanup: after completing a task (`/do`), evaluate whether each temporary file is still required. Remove unneeded files, or promote essential content into curated docs under `docs/` or into `specs/`/`plans/`.

## Stage-Gated Workflow (spec/plan/do)

- Mode: Opt-in. The workflow applies only when the user explicitly uses `/spec`, `/plan`, or `/do`. Routine Q&A or trivial edits do not require these stages.
- Triggers: A message containing one of `/spec`, `/plan`, or `/do` activates or advances the workflow. Once active, stages must proceed in order with explicit user approval to advance.
- Guardrails:
  - Do not modify source code before `/do`. Documentation/spec files may be edited only in `/spec`.
  - Do not skip stages or proceed without user confirmation once the workflow is active.
  - If scope changes, return to the appropriate prior stage for approval.
  - Respect sandbox/approval settings for all actions.

- When to Use
  - Use the workflow for new features, structural refactors, multi-file changes, or work needing traceability.
  - Skip the workflow (no triggers) for routine Q&A, diagnostics, or one-off trivial edits.

- Entry Points and Prerequisites
  - `/spec` is the canonical entry point for new efforts.
  - `/plan` requires an approved `/spec`. If unclear which spec applies, pause and ask the user to identify the correct file(s) under `specs/`.
  - `/do` requires an approved `/plan`.

- `/spec` (Specifications; docs only)
  - Purpose: Capture a concrete, reviewable specification using spec-kit style.
  - Output: Markdown spec(s) under `specs/` (no code changes). Share a concise diff summary and links to updated files; wait for approval.
  - Style: Specs are canonical and final. Do not include change logs or "correction/更正" notes. Incorporate revisions directly so the document always reflects the current agreed state. Historical context belongs in PR descriptions, commit messages, or the conversation - not in the spec.
  - Recommended contents:
    - Problem statement and context
    - Goals and non-goals
    - Requirements and constraints (functional, UX, performance, security)
    - UX/flows and API/IPC contracts (as applicable)
    - Acceptance criteria and success metrics
    - Alternatives considered and open questions
    - Rollout/backout considerations and telemetry (if relevant)

- `/plan` (High-level Plan; docs only)
  - Purpose: Turn the approved spec into an ordered, verifiable implementation plan.
  - Inputs: Approved spec file(s) in `specs/`.
  - Ambiguity: If the relevant spec is unclear, pause and request clarification before writing the plan.
  - Style: Plans are canonical and should not include change logs or "correction/更正" notes. Incorporate revisions directly so the plan always reflects the current agreed state. Historical notes should live in PR descriptions, commit messages, or the conversation.
  - Output:
    - An ordered plan via `update_plan` (short, verifiable steps; Task is merged into Plan and tracked here).
    - A plan document in `plans/` named `YYYY-MM-DD-short-title.md`, containing:
      - Scope and links to authoritative spec(s)
      - Assumptions and out-of-scope items
      - Phases/milestones mapped to acceptance criteria
      - Impacted areas, dependencies, risks/mitigations
      - Validation strategy (tests/lint/build) and rollout/backout notes
      - Approval status and next stage
  - Handoff: Await user approval of the plan before `/do`.

- `/do` (Execution)
  - Purpose: Implement approved plan steps with minimal, focused changes and file operations.
  - Actions:
    - Use `apply_patch` for file edits; group related changes and keep diffs scoped to approved steps.
    - Provide concise progress updates and a final summary of changes.
    - Validate appropriately: run `cmake --build` (or `scripts/build.ps1`), `ctest`, relevant `dotnet test` projects, and `scripts/run_verification_tests.ps1` where it makes sense.
    - If material changes to the plan are needed, pause and return to `/plan` (or `/spec`) for approval.
  - Output: Implemented changes, validation results, and a concise change summary linked to the plan checklist.

### Plans Directory

- Location: `plans/` at the repository root.
- Filename: `YYYY-MM-DD-short-title.md` (kebab-case title; consistent dating).
- Style: Plan docs are the canonical source of truth for the implementation approach; avoid embedding change logs or "correction/更正" notes. Update the plan in place as decisions evolve.
- Contents:
  - Title and summary
  - Scope and linked specs (paths under `specs/`)
  - Assumptions / Out of scope
  - Step-by-step plan (short, verifiable)
  - Validation strategy (tests/lint/build)
  - Approval status and next stage
- Process:
  - During `/plan`, create or update the relevant file in `plans/` and share a short summary in the conversation. Await approval before `/do`.
