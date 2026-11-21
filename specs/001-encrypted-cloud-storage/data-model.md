# Data Model: Encrypted Cloud Storage

**Feature**: `001-encrypted-cloud-storage` | **Date**: 2026-11-20 | **Plan**: [plan.md](plan.md)

The binary layouts are specified in [contracts/file-format.md](contracts/file-format.md). This
document defines the entities, their fields, how they relate, the validation rules, and the state
transitions.

## Entity overview

```text
KeyProfile (local, 1)  ──matches──▶  VaultDescriptor (one per remote, all identical)
     │                                        │
     │ unlocks                                │ stored at root of
     ▼                                        ▼
MasterKey ──HKDF──▶ ContentKey, NameKey, CheckKey      Remote (0..n, local config)
                                                       │ contains
                                                       ▼
LocalItem ──encrypt──▶ EncryptedObject ◀──addressed by── RemotePath
```

---

## KeyProfile

The local record of which key this computer uses. It is stored in `config.json` under `key`.
It contains no secret.

| Field | Type | Rules |
|-------|------|-------|
| `vault_id` | UUID v4 string | Required. Must equal the `vault_id` of every connected remote's descriptor. |
| `key_source` | `"passphrase"` \| `"keyfile"` | Required. |
| `kdf` | KdfParams \| null | Required when `key_source = passphrase`. Must be null otherwise. |
| `key_check` | base64 (32 bytes) | Required. `HMAC-SHA256(CheckKey, "hcd key check v1")`. |
| `keyfile_path` | absolute OS path \| null | Required when `key_source = keyfile`. Stored in the native path form. |

**KdfParams**: `alg` = `"argon2id"`, `salt` (base64, 16 bytes), `time` (≥ 3), `memory_kib`
(≥ 65536), and `threads` (1–16). Lower values are rejected when an existing descriptor is read.

**Validation**:
- A profile is valid only if deriving or reading the MasterKey reproduces `key_check`. Anything
  else is reported as a wrong passphrase or wrong key file (exit code 3).

## MasterKey and subkeys *(never persisted in plaintext)*

| Key | Size | Derivation |
|-----|------|------------|
| MasterKey | 32 B | Argon2id(passphrase NFC, KdfParams), or the key file's contents |
| ContentKey | 32 B | HKDF-SHA256(MasterKey, info=`"hcd v1 content"`) |
| NameKey | 64 B | HKDF-SHA256(MasterKey, info=`"hcd v1 names"`) |
| CheckKey | 32 B | HKDF-SHA256(MasterKey, info=`"hcd v1 check"`) |

- A MasterKey can live only in process memory or in the OS credential store (service
  `hidden-cloud-drive`, account `vault_id`).
- Key buffers are zeroed after use where the language permits.

## KeyFile

A user-held text file containing the MasterKey (key-file mode only). Its format is in
[contracts/file-format.md §3](contracts/file-format.md).

- It is created with owner-only permissions (`0600` on POSIX; on Windows, the user profile's
  default ACL).
- On import, the tool rejects files with the wrong header, the wrong length, or a key check that
  does not match.

## VaultDescriptor

This is a non-secret JSON object at each remote root (`.hcd-vault.json`). Its schema is in
[contracts/file-format.md §2](contracts/file-format.md).

- It contains the same `vault_id`, `key_source`, `kdf`, and `key_check` as the KeyProfile, plus
  `format` (`"hcd-vault"`), `version` (1), and `created` (RFC 3339 UTC).
- **Invariant**: every remote added on any computer with the same key has a byte-identical
  descriptor, apart from `created`.
- Adding a remote that already has a descriptor with a different `vault_id` is refused with
  exit code 4 (conflict). Mixing keys in one location is never allowed.

## Remote

A named connection to a storage location. It is stored in `config.json` under `remotes.<name>`.

| Field | Type | Rules |
|-------|------|-------|
| name *(map key)* | string | `^[a-z][a-z0-9_-]{1,31}$`. The 2-character minimum keeps it distinct from a Windows drive letter. |
| `type` | `"gdrive"` \| `"s3"` \| `"local"` | Required. |
| *gdrive* `client_id` | string | Required. The user's OAuth Desktop client. |
| *gdrive* `client_secret` | string | Required. For Desktop clients, Google does not treat this as confidential. |
| *gdrive* `folder_id` | string | Required. The Drive ID of the vault root folder, resolved and saved at `remote add`. |
| *gdrive* `folder_path` | string | For display only. |
| *s3* `bucket` | string | Required. |
| *s3* `prefix` | string | Optional. If not empty, it is normalized to end with `/`. |
| *s3* `region` | string | Required. |
| *s3* `profile` | string \| null | Optional. The AWS shared config profile. |
| *local* `path` | absolute OS path | Required. Must be an existing directory. |

**Secrets**: Drive refresh tokens are kept in the OS credential store, under account
`gdrive:<remote name>`. If no store is available, they go in `tokens/<name>.json` in the config
directory, readable only by the user. S3 credentials are never stored by this tool (they come from
the AWS credential chain).

## RemotePath

A user-facing path inside a remote, written `REMOTE:a/b/c.txt`.

- Segments are separated by `/`. A backslash is accepted on input on Windows and converted to `/`.
- Each segment is normalized to NFC, must not be empty, must not be `.` or `..`, and must be no
  more than 255 UTF-8 bytes.
- The encrypted form is each segment replaced by
  `base32lower(AES-SIV(NameKey, seg, AD = "hcd-name-v1" ‖ parentPath))`.
- **S3 only**: the prefix plus the encrypted key must be no more than 1024 bytes. Otherwise the
  operation fails before any upload starts.

## LocalItem

A file or directory on the user's computer that is the source of an upload or the target of a
download.

- Its relative path is computed from the root the user named, and NFC-normalized.
- Symbolic links are skipped with a warning. Special files (devices, sockets, FIFOs) are skipped
  with a warning.
- **On download**: names are made safe for the target platform (see research R13), with a warning
  printed for each renamed item.

## EncryptedObject

What is actually stored at the provider for each file.

| Attribute | Source |
|-----------|--------|
| stored name | The encrypted name of the last RemotePath segment |
| stored parent | The encrypted parent path (a Drive folder chain, or an S3 key prefix) |
| bytes | Header (8 B) followed by the Tink streaming AEAD ciphertext. See file-format §1. |
| ciphertext size | Provided by the provider. Used to compute the plaintext size for listings without downloading. |

**Classification during listing**:
- `ok`: the name decrypts.
- `foreign`: the name does not decode, or fails AES-SIV authentication. The item is shown as
  "not encrypted by hcd" and is never modified unless the user names it explicitly.
- The `.hcd-vault.json` file is hidden from listings.

## Transfer (state transitions)

A single file upload or download. This is runtime state only and is not persisted.

```text
Upload:   pending ─▶ checking-destination ─┬─▶ conflict (exists & !--overwrite) ─▶ skipped
                                           └─▶ encrypting+sending ─┬─▶ committed
                                                                   └─▶ failed ─▶ aborted (no remote object)

Download: pending ─▶ checking-destination ─┬─▶ conflict (local exists & !--overwrite) ─▶ skipped
                                           └─▶ receiving+decrypting (to temp) ─┬─▶ verified ─▶ renamed ─▶ committed
                                                                                └─▶ failed ─▶ temp deleted
```

- A `failed` transfer never leaves an object at the remote, or a file at the local target name
  (FR-004, FR-021).
- An authentication failure is reported per file:
  - `wrong key`: the vault check or the header failed.
  - `tampered or corrupted`: a segment tag failed or the file was truncated.

  In both cases the batch continues with the remaining files and exits with code 3.
