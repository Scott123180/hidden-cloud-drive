# Phase 0 Research: Encrypted Cloud Storage

**Feature**: `001-encrypted-cloud-storage` | **Date**: 2026-11-20 | **Plan**: [plan.md](plan.md)

Each decision below resolves an open item from the plan's Technical Context. Decisions are scored
against the constitution: TDD (I), simplicity (II), portability (III), and cross-platform ease
of use (IV).

---

## R1. Implementation language and distribution

**Decision**: Go (1.26 or newer), built as one statically linked `hcd` binary per OS and
architecture with `CGO_ENABLED=0`. Releases are produced by GoReleaser and published as archives,
a Homebrew formula (macOS and Linux), a Scoop manifest (Windows), and via `go install`.

**Rationale**:
- One binary with no runtime to install satisfies one-step install on every platform (IV).
- Go cross-compiles to Linux, macOS, and Windows on x86_64 and arm64 from a single machine, and
  its standard library handles paths and platform config directories (`os.UserConfigDir`) (III).
- The standard library includes a test runner, race detector, and benchmarks, so no test framework
  dependency is needed (I, II).
- First-class Google API and AWS SDK client libraries exist for Go.

**Alternatives considered**:
- *Rust*: equally portable with a stronger type system, but slower to iterate on and with a
  steeper learning curve. Its Google Drive client ecosystem is less mature.
- *Python*: fast to write, but users must install an interpreter, or the tool must be bundled
  with PyInstaller and similar tools, which is fragile and fails the one-step install rule.
- *Node/Electron*: heavy runtime, and a GUI is not required for this feature.

---

## R2. Key hierarchy and passphrase derivation

**Decision**:
- A 32-byte **master key** comes from one of two sources:
  - **Passphrase mode**: `Argon2id(passphrase_NFC, salt, t=3, m=64 MiB, p=4, len=32)`. The salt
    is 16 random bytes. These are the RFC 9106 "second recommended" parameters.
  - **Key-file mode**: 32 bytes from the operating system's cryptographically secure random
    generator (`crypto/rand`).
- Purpose-specific **subkeys** are derived from the master key with HKDF-SHA256 (RFC 5869),
  using distinct `info` labels:
  - `"hcd v1 content"`: 32 bytes, for content encryption.
  - `"hcd v1 names"`: 64 bytes, for AES-SIV name encryption.
  - `"hcd v1 check"`: 32 bytes, for the key-check value.
- The **key check** is `HMAC-SHA256(check_subkey, "hcd key check v1")`. It lets the tool reject a
  wrong passphrase or key file immediately, with a clear message (FR-004, SC-004).

**Rationale**: Argon2id is the current OWASP and RFC 9106 recommendation for password hashing
and resists GPU and ASIC guessing (FR-006a). With these parameters, derivation takes well under
1 second on a typical desktop. Domain-separated subkeys keep each primitive's key independent.
Normalizing the passphrase to Unicode NFC ensures the same passphrase yields the same key on
macOS, Windows, and Linux, where input methods may differ (FR-006a, III).

**Alternatives considered**:
- *scrypt*: acceptable, but Argon2id is the newer standard and is in `golang.org/x/crypto`.
- *PBKDF2*: does not require much memory, so it is much weaker against GPU guessing.
- *Using the master key directly for every primitive*: reusing one key across different
  algorithms is discouraged.

**Note**: The key check, like any ciphertext, lets someone holding the stored data attempt offline
guessing. The Argon2id cost and the passphrase strength rules (R9) are the mitigation. Key-file
mode is immune.

---

## R3. Where the passphrase salt lives (reproducibility across computers)

**Decision**: Each remote location's root holds a small, non-secret **vault descriptor**
(`.hcd-vault.json`). It contains the vault ID, key source, KDF parameters and salt, and the key
check. The same descriptor is written to every remote that uses the same key. A new computer
bootstraps by connecting to a remote, reading its descriptor, and asking for the passphrase or key
file (`hcd key import`).

**Rationale**: FR-006a requires the same passphrase to produce the same key on any computer. A
random salt is necessary for security but must then be retrievable. Storing it next to the data
means nothing extra has to be carried between machines. Because every remote shares the same
salt and master key, files moved between providers still decrypt (FR-011). The descriptor
contains no secret material (FR-007).

**Alternatives considered**:
- *Fixed or empty salt*: allows precomputed guessing against all users of the tool. Rejected.
- *Salt in each file header, with Argon2id run per file*: too slow (about 1 second per file).
  Rejected.
- *Salt carried only in a local config file*: the user would have to copy config files between
  machines, which fails ease of use (IV).

---

## R4. Content encryption (streaming, large files, tamper detection)

**Decision**: Use the **streaming AEAD from Google Tink, AES-256-GCM-HKDF with 1 MiB segments**
(`github.com/tink-crypto/tink-go/v2`, `streamingaead/subtle`), keyed with the content subkey. The
associated data is the file's 8-byte format header followed by the file's NFC-normalized relative
path. The full layout is in [contracts/file-format.md](contracts/file-format.md).

**Rationale**:
- This is a publicly specified, audited, and widely deployed construction (FR-002). Tink derives
  a fresh per-file key from a random salt, so there are no nonce-reuse concerns across files.
- Encryption and decryption are streamed segment by segment, so memory use is constant regardless
  of file size (FR-020, SC-002 up to 10 GB).
- Each segment is authenticated, and the final segment is marked. This detects truncation,
  reordering, and bit flips (FR-004, SC-003).
- Binding the relative path into the associated data stops a provider from swapping the contents
  of two stored files undetected.
- AES-GCM runs at GB/s on CPUs with AES hardware instructions (all mainstream x86_64 and arm64
  desktops), which comfortably meets SC-006 (no more than 10% overhead).

**Alternatives considered**:
- *Hand-rolled STREAM construction over ChaCha20-Poly1305*: a small amount of code, but it is
  security-critical code written from scratch. Using a vetted implementation is the simpler and
  safer choice (II).
- *The `age` file format*: well designed, but its passphrase mode runs scrypt for every file,
  and its library exposes only whole-file recipients.
- *Single-shot AES-GCM over the whole file*: requires the whole file in memory, and is limited to
  about 64 GB per nonce. Rejected (FR-020).

**Consequence of binding the path**: a stored file only decrypts at the same relative path. Moving
a file between providers at the same path works (FR-011). Moving it to a different folder through
the provider's web interface does not. This limitation is documented in the quickstart.

---

## R5. File and folder name encryption

**Decision**: Encrypt each path segment separately with **AES-SIV** (RFC 5297, deterministic
AEAD; Tink `daead/subtle`), using the 64-byte names subkey. The associated data is
`"hcd-name-v1" || NFC parent relative path`. The ciphertext is encoded as **lowercase base32
without padding** (RFC 4648 alphabet).

**Rationale**:
- Deterministic encryption means the same name in the same folder always produces the same
  ciphertext. The tool can therefore find a file or folder by name, and check whether a
  destination exists, without listing and decrypting everything (FR-018, FR-016).
- Using the parent path as associated data means identical names in different folders produce
  different ciphertexts, so less is revealed about the folder structure.
- Base32 lowercase uses only `[a-z2-7]`. It is safe on case-insensitive file systems and for S3
  keys and Drive names (III).
- A name that fails to decode or authenticate was not written by this tool. Listings use this to
  mark foreign files (US3 scenario 3).

**Limits**: plaintext segments are capped at 255 UTF-8 bytes, which encodes to at most 434
characters. S3 object keys are limited to 1024 bytes in total. When a path would exceed that
limit, the tool shows a clear error and creates nothing.

**Alternatives considered**:
- *Randomized name encryption with a separate index file*: hides even same-name equality, but an
  index must then be kept consistent across computers. That is a sync problem, which is out of
  scope. Rejected (II).
- *EME wide-block mode (the `rclone crypt` approach)*: not an AEAD, and no vetted Go library is
  maintained for it.
- *Hashed names*: cannot be decrypted back to the original name (fails FR-009a).

---

## R6. Caching the key on the computer

**Decision**: After a successful unlock, the master key is cached in the **OS credential store**
(macOS Keychain, Windows Credential Manager, or Secret Service on Linux) via
`github.com/zalando/go-keyring`, under service `hidden-cloud-drive` and account `<vault_id>`.
- `hcd key lock` removes the cached key.
- `--no-remember` skips caching for that unlock.
- If no credential store is available (for example, a headless Linux machine), the tool prompts
  for the passphrase or reads the key file on every command and prints a one-time notice. It never
  writes the key to a plaintext file.

**Rationale**: Prompting for an expensive passphrase on every command is a usability failure
(IV). The OS credential stores are the conventional, protected places for secrets on each platform
(FR-007). `go-keyring` is pure Go on all three platforms, so the build stays cgo-free (III).

**Alternatives considered**:
- *Always prompt*: secure, but tedious.
- *An encrypted key cache file protected by another password*: just moves the problem.

---

## R7. Google Drive integration

**Decision**:
- Client library: `google.golang.org/api/drive/v3` with `golang.org/x/oauth2`.
- Sign-in uses the **OAuth 2.0 installed-app flow with a loopback redirect and PKCE**. The tool
  opens the browser and listens on `127.0.0.1` on a random port.
- Scope: `https://www.googleapis.com/auth/drive`. The **user supplies their own OAuth client ID**,
  created once in Google Cloud Console as a "Desktop app" with the app set to "In production" so
  refresh tokens do not expire after 7 days. The quickstart walks through this.
- The refresh token is stored in the OS credential store (R6). If none is available, it is saved
  to a file in the config directory, readable only by the user, and the tool prints a warning.
- Each remote is a single Drive folder, the vault root, identified by its folder ID. Each encrypted
  path segment maps to a Drive folder or file.
- Uploads use Drive **resumable uploads**. Drive only creates the file once all bytes have been
  sent, so an interrupted upload leaves nothing behind (FR-021).

**Rationale**: The full `drive` scope is needed to see files this tool did not create, which
covers hand-moved files (US2 scenario 2) and marking foreign files (US3 scenario 3). The narrower
`drive.file` scope hides those files from the app entirely. Restricted scopes require a Google
security assessment for a shared, published client, so each user brings their own client, as
`rclone` recommends.

**Alternatives considered**:
- *`drive.file` scope with a client shipped in the tool*: no Cloud Console setup, but it breaks
  US2 scenario 2 and US3 scenario 3. It remains a fallback if that trade-off is acceptable (see
  risks in plan.md).
- *Shipping a shared, verified client*: requires a yearly third-party security assessment. Out of
  scope for a personal tool.

---

## R8. Amazon S3 integration

**Decision**:
- Use the AWS SDK for Go v2 (`github.com/aws/aws-sdk-go-v2`) with the **standard credential
  chain**: environment variables, the shared `~/.aws` config and credentials files (including
  named profiles and SSO), and instance roles.
- A remote is defined by bucket, key prefix, region, and an optional profile name.
- Large files use multipart upload through the SDK's transfer manager. On any error, the multipart
  upload is aborted (FR-021). The quickstart recommends a bucket lifecycle rule that cleans up
  incomplete multipart uploads after a crash.
- Writes use **conditional writes (`If-None-Match: *`)**, which S3 has supported since August 2024
  for both single-part and multipart uploads. This makes "never overwrite silently" (FR-018) safe
  even if two uploads race. `--overwrite` omits the condition.

**Rationale**: The standard credential chain means the tool stores no AWS secrets itself
(FR-012). The shared config files live in the conventional place on each OS (III, IV).

**Alternatives considered**:
- *Storing access keys in the tool's own config*: duplicates the AWS CLI's job and creates another
  copy of a secret to protect.
- *S3-compatible endpoints (MinIO, Backblaze, and others)*: easy to add later with an endpoint
  override. Deferred because no current requirement asks for it (II).

---

## R9. Passphrase strength rules (FR-006b)

**Decision**: A passphrase is accepted only if all of these hold:
- It is at least 12 characters (Unicode code points, after NFC normalization).
- It is not in an embedded list of the 10,000 most common passwords (compared case-insensitively).
- It is not a single character repeated, or a simple sequence.

When a passphrase is rejected, the message says which rule failed and suggests using four or more
random words.

**Rationale**: These rules are simple, deterministic, and testable (I, II). They are aligned with
the NIST SP 800-63B guidance: a minimum length, a blocklist, and no composition rules. The list
adds about 80 KB to the binary.

**Alternatives considered**:
- *zxcvbn entropy estimation*: gives better feedback, but the Go ports are unmaintained and its
  scores are harder to test deterministically.

---

## R10. Atomic writes and interrupted transfers (FR-004, FR-021)

**Decision**:
- **Downloads**: decrypt into a temporary file named `.<name>.hcd-partial-<random>` in the
  destination directory, so it is on the same file system. After the final segment authenticates,
  `fsync` the file, then rename it into place. On any error or interrupt (SIGINT on POSIX,
  Ctrl+C or Ctrl+Break on Windows), delete the temporary file.
- **Uploads**: use provider-native atomic mechanisms (Drive resumable upload, S3 single
  PutObject or multipart upload). For the local remote, write a temporary file and rename it.

**Rationale**: No plaintext is visible at the final name until the whole file is verified. Rename
within the same directory is atomic on all three operating systems (Go's `os.Rename` uses
`MoveFileEx` with replace semantics on Windows).

---

## R11. CLI framework and configuration

**Decision**:
- CLI: `github.com/spf13/cobra`. It provides consistent subcommands, generated help, and shell
  completion for bash, zsh, fish, and PowerShell.
- Hidden passphrase input: `golang.org/x/term`.
- Config: a JSON file at `os.UserConfigDir()/hidden-cloud-drive/config.json`. That resolves to
  `$XDG_CONFIG_HOME` or `~/.config` on Linux, `~/Library/Application Support` on macOS, and
  `%AppData%` on Windows. The file is parsed with `encoding/json` from the standard library.
- Remote paths use the form `REMOTE:path`. Remote names must be 2 to 32 characters
  (`^[a-z][a-z0-9_-]{1,31}$`), so a Windows drive letter such as `C:` is always treated as a
  local path (III).

**Rationale**: Cobra is the de-facto Go CLI library, and shell completion on every OS helps ease
of use (IV). JSON from the standard library avoids adding a TOML or YAML dependency (II).

**Alternatives considered**:
- *Standard library `flag` with hand-written subcommand dispatch*: fewer dependencies, but help
  and completion would have to be written and kept consistent by hand.

---

## R12. Testing strategy (Constitution I)

**Decision**:
- **Unit tests** (`go test ./...`) for each package, written before the code they cover:
  - Crypto: known-answer vectors pinned in `testdata/`, round-trip property tests, a single-byte
    tamper test at every segment boundary, truncation tests, and wrong-key tests.
- **Provider contract suite** (`internal/remote/remotetest`): one shared table of behavioral tests
  that every provider adapter must pass. It covers put, get, stat, list, delete, no-overwrite,
  and interrupted writes. It runs against:
  - `local` (a temporary directory), offline.
  - `s3` against an in-process fake S3 server (`github.com/johannesboyne/gofakes3`) on a loopback
    address, offline.
  - `gdrive` against an `httptest` fake that implements only the Drive v3 endpoints the adapter
    uses, offline.
- **End-to-end CLI tests** (`test/e2e`) build the `hcd` binary and drive it against a `local`
  remote. Prompts are fed through stdin.
- **Real-provider integration tests**: behind the `integration` build tag. They need credentials
  in environment variables, run in CI only where secrets exist, and are skipped otherwise.
- **CI matrix**: `ubuntu-latest`, `macos-latest`, and `windows-latest` run unit, contract, and
  end-to-end tests on every change. One cross-platform fixture check verifies that encrypted
  fixtures committed from each OS decrypt on every OS (SC-007).

**Rationale**: Every test is deterministic and needs no network access by default, as the
constitution's TDD principle requires (I). The contract suite keeps adapters interchangeable (III).

---

## R13. Unicode, case, and platform file name safety

**Decision**:
- Normalize all path segments to **NFC** before they are encrypted or used as associated data.
  macOS may return NFD names, while Linux and Windows usually use NFC.
- On download, rewrite names the target OS cannot store, and warn the user (US4 scenario 2):
  - Windows: the reserved characters `<>:"/\|?*`, control characters, the reserved device names
    (`CON`, `PRN`, `AUX`, `NUL`, `COM1`–`COM9`, `LPT1`–`LPT9`), and trailing dots or spaces.
    Each invalid character is replaced with `_`, and a reserved device name gets a `_` suffix.
  - Every OS: the NUL byte.
- If two remote names differ only by case and the destination file system is case-insensitive,
  the second name is treated as an existing local file (FR-018).
- Symbolic links are never followed on upload. They are skipped with a warning.

**Rationale**: Without these rules, a name that works on one OS can be unrepresentable or ambiguous
on another (III, IV).
