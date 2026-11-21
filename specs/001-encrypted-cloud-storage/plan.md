# Implementation Plan: Encrypted Cloud Storage

**Branch**: `001-encrypted-cloud-storage` | **Date**: 2026-11-20 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-encrypted-cloud-storage/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

Build `hcd`, a single-binary command-line tool for Linux, macOS, and Windows. It encrypts files
and their names on the user's computer before uploading them to Google Drive or Amazon S3, and
decrypts them after download. The cloud provider stores only ciphertext under random-looking
names.

**Key**: a 32-byte master key, either derived from a passphrase with Argon2id or generated as a
random key file. Separate subkeys are derived from it with HKDF.

**Content encryption**: Tink's audited streaming AEAD (AES-256-GCM-HKDF, 1 MiB segments). Memory
use stays constant for files of any size, and any tampering is detected.

**Name encryption**: deterministic AES-SIV per path segment, so files can be looked up by name
without an index.

**New computer**: each remote holds a non-secret vault descriptor containing the salt and a key
check. A second computer only needs the passphrase or key file.

**Providers**: each sits behind a small, contract-tested interface. Every transfer is atomic, so
an interrupted transfer never leaves a partial file.

## Technical Context

**Language/Version**: Go 1.26 or newer (research R1)

**Primary Dependencies**:
- `golang.org/x/crypto`: Argon2id and HKDF.
- `github.com/tink-crypto/tink-go/v2`: streaming AEAD and AES-SIV.
- `google.golang.org/api/drive/v3` and `golang.org/x/oauth2`: Google Drive.
- `github.com/aws/aws-sdk-go-v2`: S3.
- `github.com/zalando/go-keyring`: the OS credential store.
- `github.com/spf13/cobra`: the CLI.
- `golang.org/x/term`: hidden passphrase input.
- Test only: `github.com/johannesboyne/gofakes3`.

**Storage**:
- Local: a JSON config file in the OS config directory, plus secrets in the OS credential store.
- Remote: encrypted objects plus a `.hcd-vault.json` descriptor per remote.

**Testing**: `go test`, organized as:
- unit tests;
- a shared provider contract suite run against `local`, fake S3, and fake Drive;
- end-to-end CLI tests;
- real-provider tests behind the `integration` build tag.

**Target Platform**: Linux, macOS, and Windows on x86_64 and arm64. Builds are static with
`CGO_ENABLED=0`.

**Project Type**: CLI tool, a single Go module with one binary.

**Performance Goals**:
- Encryption overhead is no more than 10% of transfer time (SC-006).
- Content encryption runs at 500 MB/s or more on hardware with AES instructions.
- Passphrase key derivation takes under 1 second on a typical desktop.

**Constraints**:
- Memory use stays constant per transfer, under 64 MB plus the Argon2 working set, whatever the
  file size (FR-020).
- No plaintext data, plaintext names, or key material is ever sent to a provider (FR-001, FR-007,
  FR-009).
- No network access is needed for the default test suite.

**Scale/Scope**:
- A single user.
- Files from 0 bytes to at least 10 GB (SC-002).
- Thousands of files per remote.
- Transfers run one file at a time in v1.

All earlier NEEDS CLARIFICATION items are resolved in [research.md](research.md) (R1–R13).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle / Rule | How this plan complies | Pre-design | Post-design |
|------------------|------------------------|------------|-------------|
| **I. TDD (non-negotiable)** | Tests come before code in every task. The crypto has known-answer vectors, tamper and truncation tests, and golden fixtures for each format version. One contract suite (`remotetest`) gates every adapter. Every CLI contract row has an e2e test. The default suite is deterministic and offline (fakes on loopback only). | PASS | PASS |
| **II. Clean & simple** | One module, one binary, and a few small internal packages. No sync engine, index, or database. Vetted crypto libraries are used instead of hand-written code. Each dependency is justified in research.md, and the standard library is used where it is enough (for example, the JSON config). | PASS | PASS |
| **III. Portability** | Pure Go with no cgo. `os.UserConfigDir` and `filepath` are used throughout. Names are normalized to NFC. Remote names are base32 and safe on case-insensitive systems. File names are made safe for Windows. Platform differences are isolated behind `keyring` and `remote` interfaces. The formats are versioned, platform-neutral contracts. | PASS | PASS |
| **IV. Cross-platform ease of use** | One-step install via Homebrew, Scoop, `go install`, or an archive. Commands and exit codes are identical on every OS. Config lives in each OS's conventional location. Errors say what went wrong and how to fix it, with no stack traces by default. Sensible defaults (for example, the Drive folder `HiddenCloudDrive`). | PASS (risk noted) | PASS (risk noted) |
| **Constraints: CI matrix** | GitHub Actions runs on `ubuntu-latest`, `macos-latest`, and `windows-latest` for every change. | PASS | PASS |
| **Constraints: reproducible builds** | `go.mod` and `go.sum` are committed, and GoReleaser builds are reproducible (`-trimpath`, fixed `mod_timestamp`). | PASS | PASS |
| **Workflow: tests before code** | `/speckit-tasks` MUST order test tasks before implementation tasks. | PASS | PASS |

**Risk noted under IV**: Google Drive requires the user to create their own OAuth client in Google
Cloud Console, which takes about 5 minutes (research R7). This is the main threat to SC-005, the
10-minute first run. S3 and local remotes are not affected. A fallback is available if this proves
too slow: the `drive.file` scope with a client shipped in the tool. That would give up hand-moved
files (US2 scenario 2) and foreign-file display (US3 scenario 3) on Drive. No principle is violated,
so no Complexity Tracking entry is needed.

**Gate result**: PASS, with no unjustified violations.

## Project Structure

### Documentation (this feature)

```text
specs/001-encrypted-cloud-storage/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R1–R13
├── data-model.md        # Phase 1: entities, validation, state transitions
├── quickstart.md        # Phase 1: build/test + validation scenarios S1–S10
├── contracts/
│   ├── cli.md           # Commands, flags, output, exit codes
│   ├── file-format.md   # Encrypted file, vault descriptor, key file, encrypted names (v1)
│   └── provider.md      # Internal storage adapter interface + contract-suite rules
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (/speckit-tasks — NOT created here)
```

### Source Code (repository root)

```text
cmd/
└── hcd/
    └── main.go                 # Wires the CLI; no logic

internal/
├── crypto/                     # Pure functions, no I/O beyond io.Reader/Writer
│   ├── keys.go                 # Argon2id, HKDF subkeys, key check
│   ├── content.go              # Header + Tink streaming AEAD encrypt/decrypt, size math
│   ├── names.go                # AES-SIV per segment, base32, NFC
│   ├── passphrase.go           # Strength rules + embedded common-password list
│   └── testdata/               # Known-answer vectors, golden v1 fixtures
├── vault/                      # KeyProfile, VaultDescriptor, KeyFile parse/write,
│                               # unlock/lock via credstore
├── credstore/                  # OS credential store interface: keyring impl + fallback + fake
├── config/                     # config.json load/save, platform dirs, remote validation
├── remote/
│   ├── remote.go               # Provider interface, Entry, sentinel errors
│   ├── remotetest/             # Shared contract test suite (provider.md)
│   ├── local/                  # Local/mounted directory adapter
│   ├── s3/                     # S3 adapter (AWS SDK v2)
│   └── gdrive/                 # Drive adapter + OAuth loopback/PKCE flow
├── transfer/                   # put/get/ls/rm orchestration: path encryption, overwrite
│                               # checks, temp-file+rename, progress, platform-safe names
└── cli/                        # cobra commands, prompts, output formatting, exit codes

test/
└── e2e/                        # Builds hcd, runs contracts/cli.md scenarios on a local remote

.github/workflows/ci.yml        # Lint + test matrix (Linux/macOS/Windows)
.goreleaser.yaml                # Cross-platform release artifacts, Homebrew, Scoop
go.mod / go.sum
```

**Structure Decision**: a single Go module with Go's conventional `cmd/` and `internal/` layout.
Unit and contract tests sit next to their packages (`*_test.go`), as Go convention expects. The
end-to-end tests live in `test/e2e`.

The dependencies flow in one direction: `cli → transfer → {crypto, vault, remote} → stdlib/SDKs`.
This keeps `crypto` and `transfer` testable without any provider. Platform-specific code is
confined to `credstore`, `transfer` (local name sanitizing), and the OAuth browser launch inside
`remote/gdrive`.

## Complexity Tracking

No constitution violations. Two choices are recorded for transparency:

| Choice | Why Needed | Simpler Alternative Rejected Because |
|--------|------------|-------------------------------------|
| A `local` remote type that users can choose | It makes the whole system testable end to end, offline, on every OS (I). It is also useful for USB drives and mounted shares. | A test-only fake would not let users or CI run the quickstart offline. The adapter is about 100 lines. |
| A Tink dependency for crypto | It provides audited streaming AEAD and AES-SIV (FR-002, FR-020). | Hand-rolling STREAM chunking or SIV is a larger and riskier surface than one well-maintained dependency. |
