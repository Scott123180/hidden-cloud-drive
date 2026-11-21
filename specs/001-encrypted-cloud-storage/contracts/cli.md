# Contract: `hcd` Command-Line Interface (v1)

This is the user-facing contract. The commands, flags, output, and exit codes behave identically
on Linux, macOS, and Windows (FR-022). End-to-end tests in `test/e2e` assert every row below.

## Conventions

- **Remote path**: `REMOTE:path`, for example `gd:photos/beach.jpg`. `REMOTE:` alone means the
  vault root. Any argument without `NAME:`, where `NAME` has 2 or more characters, is a local path.
  This means `C:\Users\me` is local.
- **Streams**: results go to stdout. Progress, warnings, and errors go to stderr. Progress is shown
  only when stderr is a terminal, and is suppressed by `--quiet`.
- **Prompts**: passphrases are read from the terminal with echo off. In tests and scripts the
  passphrase is read from stdin when `--passphrase-stdin` is given. Environment variables are never
  used, because they leak into process listings.
- **Global flags**: `--config <path>` (overrides the config file location), `--quiet`,
  `--no-color`, `--help`, and `--version`.
- **Error format**: `hcd: <what went wrong>` followed by `  hint: <corrective action>`. Stack
  traces appear only with `--debug`.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Unexpected or internal error |
| 2 | Usage error (bad arguments or flags) |
| 3 | Wrong key, tampered or corrupted data, or an unsupported format version |
| 4 | Conflict: the destination exists, or the vault IDs do not match |
| 5 | Provider error: authentication, quota, or network |

When a batch operation has mixed results, the highest code that occurred is used.

## Commands

### Key management

| Command | Behavior |
|---------|----------|
| `hcd key init --passphrase` | Prompts for a new passphrase twice and enforces the strength rules (FR-006b). Derives the key, saves the KeyProfile, and shows the lost-key warning and backup advice (FR-008). Fails with code 4 if a key already exists. |
| `hcd key init --keyfile <path>` | Generates a random key and writes the key file to `<path>` with owner-only permissions. Refuses if the file exists. Saves the KeyProfile and shows the warning and backup advice. |
| `hcd key import --from <remote>` | For a new computer. Reads the remote's descriptor, then prompts for the passphrase, or with `--keyfile <path>` reads the key file. Verifies the key check and saves the KeyProfile. Fails with code 3 on a mismatch. |
| `hcd key unlock [--no-remember]` | Verifies the key and caches it in the OS credential store. Other commands unlock on demand, so running this is optional. |
| `hcd key lock` | Removes the cached key from the OS credential store. |
| `hcd key status` | Prints the vault ID, the key source, whether the key is cached, and the key file path if there is one. Never prints key material. |

### Remotes

| Command | Behavior |
|---------|----------|
| `hcd remote add <name> --type local --path <dir>` | Adds a local or mounted folder as a remote. |
| `hcd remote add <name> --type s3 --bucket <b> --region <r> [--prefix <p>] [--profile <aws-profile>]` | Checks access to the bucket using the AWS credential chain. |
| `hcd remote add <name> --type gdrive --client-id <id> --client-secret <s> [--folder <path>]` | Opens the browser for consent, using the loopback redirect with PKCE. Creates or finds the vault folder (default `HiddenCloudDrive`). Stores the refresh token in the OS credential store. |
| *(all `remote add` forms)* | If the remote has no descriptor and a key exists locally, the tool writes the descriptor. If the remote has a descriptor with the same vault ID, the remote is accepted. If the vault ID differs, the command exits with code 4. If no key exists locally but the remote has a descriptor, the tool prints a hint to run `hcd key import --from <name>`. |
| `hcd remote list` | Shows each remote's name, type, and location. With `--json`, prints an array. |
| `hcd remote remove <name>` | Forgets the remote locally and deletes its stored token. Never deletes remote data. |
| `hcd remote reconnect <name>` | Re-runs the provider sign-in. Use this after code 5 with an authentication hint. |

### Files

| Command | Behavior |
|---------|----------|
| `hcd put <local>... <remote:dir/> [--overwrite]` | Encrypts and uploads files. Directories are uploaded recursively, keeping relative paths (FR-015). A target ending in `/` is a directory. A single file uploaded to a target without a trailing `/` is renamed to that target. Existing destinations are skipped with a warning and exit code 4 unless `--overwrite` is given (FR-018). |
| `hcd get <remote:path>... <local-dir-or-file> [--overwrite]` | Downloads and decrypts files. Remote directories are downloaded recursively. Writes go through a temporary file followed by a rename (FR-004, FR-021). Names that are invalid on this OS are rewritten, with a warning (US4 scenario 2). |
| `hcd ls [remote:path] [-r] [--json]` | Lists the original names, plaintext sizes, and modification times (FR-016). Items that fail name decryption are shown as `[not encrypted by hcd] <stored name>`. JSON fields: `name`, `path`, `type` (`file` or `dir`), `size`, `modified`, `foreign` (bool). With no argument, lists the remotes. |
| `hcd rm <remote:path>... [-r]` | Deletes files. Folders require `-r` (FR-017). Foreign items are deleted only when named by their exact stored name. |

### Other

| Command | Behavior |
|---------|----------|
| `hcd version` | Prints the version, commit, OS and architecture, and the supported format versions. |
| `hcd completion <bash\|zsh\|fish\|powershell>` | Prints a shell completion script. |

## Example session (illustrative)

```text
$ hcd key init --passphrase
New passphrase: ********
Repeat passphrase: ********
✓ Key created (vault 3f2b6c1e).
! If you forget this passphrase, your files CANNOT be recovered by anyone.
  Write it down and store it somewhere safe and offline.

$ hcd remote add aws --type s3 --bucket my-private-bucket --region us-east-1
✓ Remote "aws" added (s3://my-private-bucket/). Vault descriptor written.

$ hcd put ./taxes aws:archive/
↑ archive/taxes/2026-return.pdf  2.1 MB  ✓
↑ archive/taxes/receipts.zip     88.4 MB ✓
2 files uploaded.

$ hcd ls aws:archive/taxes
2026-return.pdf    2.1 MB  2026-11-20 18:10
receipts.zip      88.4 MB  2026-11-20 18:10
```
