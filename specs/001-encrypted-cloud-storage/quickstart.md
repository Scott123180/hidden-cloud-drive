# Quickstart & Validation Guide: Encrypted Cloud Storage

This guide shows how to build `hcd`, run the tests, and check the feature end to end. Command
details are in [contracts/cli.md](contracts/cli.md), and formats are in
[contracts/file-format.md](contracts/file-format.md).

## Prerequisites

- Go 1.26 or newer, on Linux, macOS, or Windows.
- Optional, only for the real-provider scenarios:
  - **Amazon S3**: an AWS account and a bucket you own. Configure credentials with `aws configure`
    or `aws sso login`, which writes them to `~/.aws`.
  - **Google Drive**: a Google account and a one-time OAuth client setup (see
    [Google Drive setup](#google-drive-setup)).

## Build and test

```sh
go build -o hcd ./cmd/hcd            # on Windows the output is hcd.exe
go test ./...                        # unit, contract (local/fake s3/fake drive), and e2e tests, all offline
go test -race ./...                  # race detector
go test -tags integration ./...      # real providers; needs HCD_IT_* env vars, otherwise skipped
```

Expected result: every package passes, and no test makes a network call outside loopback.

## Validation scenarios

The scenarios use a `local` remote first, so they run offline. Repeat them with `s3` and `gdrive`
remotes to validate the real providers.

In each scenario, `orig.bin` is any test file. Create a large one with
`head -c 1G /dev/urandom > orig.bin`, or on Windows
`fsutil file createnew orig.bin 1073741824`.

### S1 — Key setup and the lost-key warning (FR-006, FR-006b, FR-008)

1. Run `hcd key init --passphrase` and enter `password123`.
   **Expect**: rejected because it is too short and too common, with a hint, and exit code 2.
2. Run it again with a 4-word passphrase.
   **Expect**: `✓ Key created`, the "cannot be recovered" warning, and exit code 0.

### S2 — Round trip with unreadable storage (US1, SC-001, SC-002)

1. `mkdir vault && hcd remote add box --type local --path ./vault`
2. `hcd put orig.bin notes.txt box:docs/`
3. Inspect `./vault` directly, as the provider would see it.
   **Expect**:
   - Only `.hcd-vault.json` and random-looking base32 names. No `docs`, `orig`, or `notes`
     strings appear (`grep -r notes vault` finds nothing).
   - The file bytes start with `HCDF`, and the original content does not appear in them.
4. `hcd get box:docs/orig.bin out.bin`, then compare the files.
   **Expect**: the files are byte-identical. Use `cmp orig.bin out.bin`, or on Windows
   `fc /b orig.bin out.bin`.

### S3 — Wrong key (US1 scenario 3, SC-004)

1. On a second config, run `hcd --config other.json key init --passphrase` with a different
   passphrase.
2. Run `hcd --config other.json remote add box --type local --path ./vault`.
   **Expect**: exit code 4, vault mismatch.
3. Import the vault with the wrong passphrase:
   `hcd --config other2.json key import --from box`.
   **Expect**: exit code 3, "wrong passphrase", and no profile is saved.

### S4 — Tamper detection (US1 scenario 4, SC-003)

1. Flip one byte in the middle of the stored file for `docs/orig.bin` inside `./vault`.
2. Run `hcd get box:docs/orig.bin t.bin`.
   **Expect**: exit code 3, "tampered or corrupted", no `t.bin` file, and no `.hcd-partial`
   file left behind.

### S5 — Listing, foreign files, and delete (US3)

1. Copy any plain file into `./vault`.
2. Run `hcd ls -r box:`.
   **Expect**: original names and sizes, plus one `[not encrypted by hcd]` row.
3. Run `hcd rm box:docs/notes.txt`, then `hcd ls box:docs`.
   **Expect**: `notes.txt` is gone.

### S6 — No silent overwrite and interrupted transfers (FR-018, FR-021)

1. Run `hcd put orig.bin box:docs/` again.
   **Expect**: a skip warning and exit code 4. With `--overwrite`, the upload succeeds.
2. Start `hcd get box:docs/orig.bin big.bin` and press Ctrl+C halfway through.
   **Expect**: neither `big.bin` nor any `.hcd-partial-*` file exists.

### S7 — Moving files between providers (US2, FR-011)

1. Add `aws` (s3) and `gd` (gdrive) remotes using the same key.
2. Run `hcd put orig.bin aws:x/`.
3. Copy the stored object by hand to the **same relative path** inside the Drive vault folder.
   Use `aws s3 cp` and the Drive web UI, keeping the stored names unchanged.
4. Run `hcd get gd:x/orig.bin y.bin`.
   **Expect**: the result is byte-identical to the original.
   *(Note: a stored file only decrypts at the same relative path, because the path is bound into
   the encryption. See research R4.)*

### S8 — Another computer or OS (US4, SC-007)

1. On a different OS, build `hcd`, then run `hcd remote add` for the same S3 bucket or Drive
   folder.
   **Expect**: a hint to run `hcd key import`.
2. Run `hcd key import --from aws` with the same passphrase, then `hcd get aws:x/orig.bin .`.
   **Expect**: the files are byte-identical.
3. On Linux or macOS, upload files named `report?.txt` and `AUX`. Then download them on Windows.
   **Expect**: they are saved as `report_.txt` and `AUX_`, with a rename warning for each.

### S9 — Performance overhead (SC-006)

Time a 1 GB `hcd put` and `hcd get` against `aws s3 cp` of the same file over the same
connection. **Expect**: `hcd` takes no more than 110% of the `aws s3 cp` time.

### S10 — First-run time (SC-005)

Give a new user only this guide and a fresh machine with an AWS profile. Time them from
`hcd key init` to their first successful `hcd put`. **Expect**: under 10 minutes.

## Google Drive setup

This is a one-time setup, about 5 minutes.

1. In Google Cloud Console, create a project and enable the **Google Drive API**.
2. Under **OAuth consent screen**, choose the External user type. Add yourself as a user, then
   click **Publish app** to put it in production. Without this, sign-ins expire after 7 days.
   Google shows an "unverified app" screen for personal clients; click *Advanced → continue*.
3. Under **Credentials**, create an **OAuth client ID** of type **Desktop app**, and copy the ID
   and secret.
4. Run `hcd remote add gd --type gdrive --client-id <id> --client-secret <secret>`.

## S3 housekeeping (recommended)

Add a bucket lifecycle rule that runs "Delete expired object delete markers or incomplete
multipart uploads" after 1 day. This cleans up upload parts left behind if the computer crashes
mid-upload.
