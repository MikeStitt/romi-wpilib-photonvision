# romi-wpilib-photonvision Constitution

This Constitution is **authoritative** for development practices in this
repository. It supersedes ad-hoc conventions, verbal agreements, and any
conflicting guidance in `CLAUDE.md`, `AGENTS.md`,
`.github/copilot-instructions.md`, or sub-directory READMEs — those files are
thin pointers back here.

**These rules apply to every task unless explicitly overridden.** Bias toward
caution over speed on non-trivial work; use judgment on trivial tasks.

This file is the **always-read core**: the Working Rules below plus Engineering
Discipline. Detailed stack and integration knowledge lives in
[`parts/`](#parts--read-only-what-your-task-needs) — read **only** the part(s)
for the work type you are touching.

## What this repository is

This repository builds **bootable Romi robot images** for Raspberry Pi 4/5 and
Orange Pi 5, integrating **WPILib 2026/2027** with **PhotonVision 2026/2027**.

The output: `.img.xz` files that flash to microSD, boot on hardware, and provide:
- PhotonVision web UI (camera, AprilTag, ML inference)
- Romi dashboard (network config, 32U4 firmware flash, system monitoring)
- NetworkTables 4.x WebSocket bridge to desktop robot code/simulator
- AP mode with default credentials (`WPILibPi-<serial>` / `WPILib2026!`)

**PhotonVision** is an upstream project we install and configure via its
`photon-image-modifier` approach (base OS image → modification scripts → output
image). We do not fork PhotonVision; we configure it via generated config and
add Romi-specific layers.

**WPILib** provides the robot control framework (NT4, cameraserver, wpilibj).
We use its Java libraries via Gradle for the Romi dashboard (Javalin + Vue).

Because this tooling writes **bootable OS images** that run on **physical
hardware**, the defining discipline is doing so **non-destructively,
idempotently, and transparently** — a script pipeline that advises (prints
commands) by default and executes only under explicit flags, with full
traceability of versions and git SHAs.

## Working Rules

The behavioral contract. Numbered for reference, not priority.

1. **Think before coding.** State assumptions explicitly. If uncertain, ask
   rather than guess. Push back when a simpler approach exists. Stop when
   confused.
2. **Simplicity first.** Write the minimum code that solves the problem.
   Nothing speculative; no features beyond what was asked; no abstraction for
   single-use code. Three similar lines beat a premature abstraction. If a
   simpler alternative exists, choose it unless you can document why not.
3. **Surgical changes.** Touch only what the task requires. Clean up only your
   own mess. Don't "improve" adjacent code, comments, or formatting. Match the
   existing style.
4. **Read before you write.** Before adding code, read the file's exports, its
   immediate callers, and the shared utilities it would use — so you don't
   duplicate what already exists. If unsure why code is shaped a certain way,
   ask. "Looks orthogonal" is a dangerous assumption.
5. **Goal-driven execution.** Define success criteria and loop until verified.
   Don't blindly follow rigid steps; define what success looks like and iterate
   toward it.
6. **Non-destructive, idempotent image builds.** Installing or adjusting
   PhotonVision/WPILib on a target image MUST be safe to re-run and MUST
   converge to the same result. Never clobber project-specific files. Prefer a
   dry-run path, make surgical edits, and only ever touch a target image under
   version control so the human can review and revert. Do not fork upstream
   behavior — we configure via generated config and install glue only.
7. **Small, bounded, side-effect-free.** Favor small composable functions with
   explicit inputs/outputs and clear boundaries; avoid god scripts. Keep core
   logic pure; I/O (filesystem, network, spawning `qemu`, `dd`, `systemctl`)
   lives in thin, mockable wrappers. Put validation at the boundaries (CLI args,
   base image state, external commands), not for impossible internal states.
8. **Fail loud.** "Completed" is wrong if anything was skipped silently. "Tests
   pass" is wrong if any were skipped or pass for the wrong reason. Surface every
   skipped file, refused overwrite, and missing dependency. Default to surfacing
   uncertainty, never hiding it.
9. **Checkpoint long operations.** After each significant step in a multi-step
   task, summarize what was done, what is verified, and what is left. Don't
   continue from a state you can't describe back.
10. **Mind the budget.** On non-trivial work, watch the token/time budget. If a
    task is spiraling (e.g. debugging the same error repeatedly), stop,
    summarize, and restart fresh rather than overrun silently.
11. **Verify before done.** Quality gates MUST be green before you declare a task
    complete. Tests are a first-class artifact (see
    [`parts/testing.md`](parts/testing.md)).

## Engineering Discipline

### Quality Gates

All code MUST pass quality gates before being committed. For this project:

- **Shell scripts**: `shellcheck` (lint), `shfmt` (format)
- **Python scripts**: `ruff` (lint), `ruff format --check` (format), `mypy` (types), `pytest` (tests)
- **Java/Gradle**: `./gradlew check` (spotless, compile, test)
- **Image builds**: QEMU boot test + smoke test (SSH, PhotonVision UI, Romi dashboard)
- **Documentation**: ADRs updated for architectural decisions

The gate is defined in `build.ninja` / `Makefile` / CI and runs in the project's
locked dependencies (uv for Python, Gradle wrapper for Java).

- Pre-commit hooks MUST remain active; never bypass them with `--no-verify`
- CI reproduces these checks on every pull request

### Configuration is Code

Infrastructure and configuration (build scripts, `build.ninja`, `Makefile`,
Gradle configs, `photon-image-modifier` scripts, generated configs, the install
logic itself) are code, and a change to them is not done until it has been
**verified by exercising it**, not merely edited. Define the success criterion
as observed behavior and run the config to confirm it: feed a deliberately-bad
base image through the pipeline and watch it reject; run `build-romi-image.sh`
against a throwaway target and confirm it converges; re-run it and confirm it is
a no-op. Quality gates do not exercise every config (they do not run a full
hardware flash), so a silently broken config can pass it — close that gap by
hand.

### Branch & Push Policy

- **Branching**: Do work on a feature branch — never commit directly to `main`.
  Before staging the first change of any task, check
  `git branch --show-current`; if it returns `main`, run
  `git switch -c <kebab-case-name>`. One branch per logical unit.
- **Stack on unmerged work; don't force independence.** When a task builds on, or
  will touch the same files as, a branch/PR that has not merged yet, branch from
  *that branch* rather than `main`, and name the base in your summary. Do **not**
  rebase or re-create an already-stacked branch onto `main` to make its diff look
  "pure" — that is exactly what re-introduces the merge conflicts (lockfiles like
  `uv.lock`, `gradle.lockfile`, `CHANGELOG.md`, shared modules) that stacking
  avoids. Reserve independent branches off `main` for work that is genuinely
  unrelated *and* touches disjoint files. When the right base is unclear, ask
  before branching.
- **The user owns the merge order**; you only push your branch. Do not merge PRs
  on the user's behalf.
- **Pushing**: Once quality gates are locally green and you're confident CI will
  pass, `git push -u origin <branch>` without asking. Never push to `main`;
  never force-push or rewrite published history without an explicit request.
- **Merging back**: via PR only.

### Conventional Commits

All commits MUST follow `<type>(<scope>): <subject>`.

- **Types**: `feat`, `fix`, `docs`, `chore`, `style`, `test`, `build`, `ci`,
  `refactor`, `perf`, `revert`.
- **Imperative mood**, **lowercase start** (unless proper noun/acronym).
- **Subject length**: ≤80 characters. **Body wrap**: at 80 characters.
- **Atomic commits**: one logical change per commit.
- `git-cliff` generates the changelog from these commits; commits are the source
  of truth.

### Documentation Hygiene

Any behavior-affecting change MUST update affected `--help` text, READMEs, and
related documentation in the same commit. A documentation gap is a bug.
Architecture Decision Records (ADRs) in `.docs/architecture/` are the source of
truth for design decisions — update them when decisions change.

### Plan-File Etiquette

Plan files (`.docs/integration-plan.md` and session-scoped planning
artefacts) are accumulated session memory. When entering plan mode for a task
that doesn't match existing plan content: **archive** the existing plan in place
(prefix its top heading with `# Archived plan (YYYY-MM-DD): <old title>`, keep
the body), **append** the new plan to the same file, and **never** overwrite
wholesale. Starting fresh is the user's decision.

### Upstream Provenance

PhotonVision and WPILib are upstream projects we install and configure, not code
we own. When this tooling vendors or copies upstream artefacts into a target
image, keep upstream license headers intact, do not edit upstream definitions in
place (configure via generated config instead), and record which upstream version
a target was built from so re-runs and updates are traceable.

### Scratch / Probe Scripts — Explicit Escape Hatch

Throwaway scripts written to investigate upstream internals or a target image
SHOULD NOT be held to the standards above. They live in `scratch/` (gitignored),
excluded from lint / format / tests; the hooks SHOULD NOT block on them.
**Promotion**: if a probe script proves repeatedly useful, port it into
`src/` or `scripts/` with full standards applied — do not let useful logic rot
in `scratch/`. This exception is named explicitly so future contributors don't
"tidy" it away.

## Parts — read only what your task needs

| Work type                              | Read                                                                     |
| -------------------------------------- | ------------------------------------------------------------------------ |
| Shell/Python image builder scripts     | [`parts/image-builder.md`](parts/image-builder.md)                       |
| Java/Gradle Romi dashboard (Javalin)   | [`parts/romi-dashboard.md`](parts/romi-dashboard.md)                     |
| PhotonVision install/configure         | [`parts/photonvision.md`](parts/photonvision.md)                         |
| WPILib/NT4 integration                 | [`parts/wpilib.md`](parts/wpilib.md)                                     |
| Hardware testing / QEMU                | [`parts/hardware-testing.md`](parts/hardware-testing.md)                 |
| Writing tests                          | [`parts/testing.md`](parts/testing.md)                                   |
| Amending the constitution itself       | [`parts/constitution-maintenance.md`](parts/constitution-maintenance.md) |

## Governance

- **Compliance**: all pull requests and code reviews MUST verify adherence to
  these principles. Violations MUST be flagged and resolved before merge.
- **Amendment workflow & changelog**: the step-by-step amendment plan and the
  full dated version history live in
  [`parts/constitution-maintenance.md`](parts/constitution-maintenance.md).
  Read that part before changing this file or any other part.

**Version**: 1.0.0 | **Ratified**: 2026-06-18 | **Last amended**: 2026-06-18