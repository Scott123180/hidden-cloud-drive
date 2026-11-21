# Hidden Cloud Drive Constitution

## Core Principles

### I. Test-Driven Development (NON-NEGOTIABLE)

- Every change to behavior MUST begin with a failing automated test that expresses the
  intended behavior. Production code MUST NOT be written before a corresponding test exists
  and has been observed to fail.
- The Red → Green → Refactor cycle MUST be followed: write a failing test, write the minimum
  code to pass it, then refactor with the test suite green.
- Bug fixes MUST include a regression test that reproduces the bug before the fix is applied.
- Tests MUST be deterministic, isolated, and fast enough to run locally on every change. Tests
  MUST NOT depend on network access, wall-clock time, or machine-specific paths unless they are
  explicitly marked as integration tests.
- The full test suite MUST pass on every supported platform before a change is merged.

**Rationale**: Tests written first define behavior unambiguously, keep the design testable, and
provide the safety net that makes simplicity-driven refactoring and cross-platform work safe.

### II. Clean & Simple Code

- The simplest design that satisfies the current, specified requirements MUST be chosen (YAGNI).
  Speculative abstractions, extension points, and configuration options MUST NOT be added
  without a present requirement.
- Each module, function, and type MUST have a single clear responsibility and an intention-
  revealing name.
- Functions SHOULD be short and readable top-to-bottom; deep nesting and hidden side effects
  MUST be refactored away.
- Duplication MUST be removed once a pattern appears a third time; earlier extraction requires
  justification.
- New third-party dependencies MUST be justified by a clear benefit that outweighs their
  maintenance, security, and portability cost.
- Any intentional added complexity MUST be recorded and justified in the plan's Complexity
  Tracking section.

**Rationale**: Simple code is easier to test, review, port, and change. Complexity is a cost paid
on every future change and MUST earn its place.

### III. Portability

- Core logic MUST be platform-agnostic and MUST NOT call OS-specific APIs directly.
  Platform-specific behavior MUST be isolated behind small, well-defined interfaces with one
  implementation per platform.
- File paths MUST be built with platform-aware path utilities; hard-coded separators, drive
  letters, and absolute paths are prohibited.
- Code MUST NOT assume a particular filesystem case sensitivity, line ending, default text
  encoding (UTF-8 MUST be explicit), shell, or locale.
- Build, test, and run commands MUST work on every supported platform without requiring a
  POSIX shell on Windows or platform-specific tooling not documented as a prerequisite.
- Data formats written to disk or sent over the network MUST be platform-neutral and versioned.

**Rationale**: Portability is far cheaper to preserve from the start than to retrofit. Isolating
platform differences keeps the core simple and testable everywhere.

### IV. Cross-Platform Ease of Use

- Installation MUST be achievable in a single documented step per supported platform, with no
  manual dependency wrangling beyond documented prerequisites.
- The application MUST behave consistently across platforms: the same commands, options,
  defaults, and outputs, differing only where a platform convention requires it (e.g., config
  and data directory locations).
- Configuration and data MUST be stored in each platform's conventional locations
  (e.g., XDG directories on Linux, Application Support on macOS, AppData on Windows).
- Sensible defaults MUST allow the common workflow to succeed with zero configuration.
- Error messages MUST be human-readable, state what went wrong, and suggest a corrective action.
  Stack traces MUST NOT be shown to end users by default.

**Rationale**: Users choose a tool that "just works" on the machine they have. Consistent behavior
across platforms reduces support burden and documentation size.

## Platform & Technology Constraints

- **Supported platforms**: Linux, macOS, and Windows (current mainstream releases on x86_64 and
  arm64 where the toolchain allows). Adding or dropping a platform requires a constitution
  amendment.
- **Toolchain selection**: Languages, frameworks, and libraries MUST be chosen for first-class
  support on all supported platforms. A dependency that works on only a subset of platforms MUST
  be isolated behind a platform interface per Principle III.
- **Continuous integration**: CI MUST build and run the full test suite on every supported
  platform for every change.
- **Reproducible builds**: Dependency versions MUST be pinned via lockfiles or equivalent.

## Development Workflow & Quality Gates

- **Spec first**: Features follow the Spec Kit flow (specify → plan → tasks → implement). Plans
  MUST pass the Constitution Check against Principles I–IV before implementation begins.
- **Tests-first task ordering**: In every task list, test tasks MUST precede the implementation
  tasks they cover.
- **Merge gates**: A change MUST NOT be merged unless:
  1. All tests pass on every supported platform in CI.
  2. New or changed behavior is covered by tests written before the implementation.
  3. Linting and formatting checks pass.
  4. A review confirms compliance with this constitution, with any deviation justified in
     writing.
- **Small changes**: Changes SHOULD be small and focused on a single concern to keep review and
  testing simple.

## Governance

- This constitution supersedes all other project practices and guidance. Where another document
  conflicts with it, this constitution prevails.
- **Amendments** MUST be proposed in writing (via pull request modifying this file), state the
  rationale and impact on existing code and templates, and include any required migration plan.
- **Versioning** follows semantic versioning:
  - MAJOR: backward-incompatible removal or redefinition of a principle or governance rule.
  - MINOR: a new principle or section, or materially expanded guidance.
  - PATCH: clarifications, wording, and non-semantic fixes.
- **Compliance review**: Every plan and every pull request review MUST verify compliance with the
  Core Principles. Unjustified complexity or platform-specific leakage into core logic is grounds
  for rejection.

**Version**: 1.0.0 | **Ratified**: 2026-11-20 | **Last Amended**: 2026-11-20
