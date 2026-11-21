# Contract: On-Disk and On-Remote Formats (v1)

These formats are a **public, versioned contract** (FR-005). Any change to the bytes written MUST
bump the relevant version, and readers MUST keep decrypting every earlier version. Golden
fixtures for each version are committed under `internal/crypto/testdata/` and MUST decrypt in CI
on every platform.

All multi-byte integers are big-endian. "NFC" means Unicode Normalization Form C.

## 1. Encrypted file (content)

```text
offset  size  field
0       4     magic            = 0x48 0x43 0x44 0x46  ("HCDF")
4       1     format_version   = 0x01
5       1     content_alg      = 0x01  (AES256-GCM-HKDF, 1 MiB segments)
6       2     reserved         = 0x0000  (readers MUST reject non-zero in v1)
8       ...   tink_stream      = Tink streaming AEAD ciphertext (its own header + segments)
```

- **Key**: ContentKey (32 B). See [data-model.md](../data-model.md).
- **Tink parameters**: HKDF hash SHA256, derived key size 32, ciphertext segment size
  1,048,576 bytes, first segment offset 0.
- **Associated data**: `header[0:8] ‖ UTF-8(NFC(relative_path))`. Here `relative_path` is the
  plaintext RemotePath without the remote name, with `/` separators and no leading slash, for
  example `photos/2026/beach.jpg`.
- **Plaintext size from ciphertext size**: this is computed from the Tink segment layout, so
  listings can show the original size without downloading (US3).
- **Reader rules**:
  - Unknown magic: not an hcd file.
  - Unknown `format_version` or `content_alg`: error "created by a newer version of hcd".
  - Any authentication failure, including truncation: tamper or corruption error, and no
    output is produced.

## 2. Vault descriptor (`.hcd-vault.json` at each remote root)

This file is UTF-8 JSON and contains no secrets.

```json
{
  "format": "hcd-vault",
  "version": 1,
  "vault_id": "3f2b6c1e-8a4d-4f7e-9b0a-2c5d7e8f9a10",
  "key_source": "passphrase",
  "kdf": {
    "alg": "argon2id",
    "salt": "base64-16-bytes",
    "time": 3,
    "memory_kib": 65536,
    "threads": 4
  },
  "key_check": "base64-32-bytes",
  "created": "2026-11-20T18:04:05Z"
}
```

- When `key_source` is `"keyfile"`, `kdf` is `null`.
- `key_check` = `HMAC-SHA256(CheckKey, "hcd key check v1")`.
- Readers MUST reject a `version` they do not know, and MUST reject KDF parameters below the
  minimums in data-model.md. This stops a malicious provider from downgrading the cost.

## 3. Key file (key-file mode, held by the user)

The key file is UTF-8 text with LF or CRLF line endings, both accepted.

```text
# hidden-cloud-drive key file. KEEP SECRET. Anyone with this file can read your data.
# If you lose it, your data cannot be recovered. Back it up somewhere safe and offline.
hcd-key-v1:<base64url, no padding, 32 bytes = 43 chars>
```

- Lines starting with `#` and blank lines are ignored. Exactly one `hcd-key-v1:` line is required.

## 4. Encrypted names

For each path segment:

```text
enc_segment = base32_lower_nopad( AES-SIV(NameKey, NFC(segment), AD = "hcd-name-v1" ‖ NFC(parent_path)) )
```

- `parent_path` is the plaintext relative path of the containing folder. It is an empty string at
  the vault root.
- Base32 uses the RFC 4648 alphabet, lowercased (`a-z2-7`).
- Plaintext segments are at most 255 UTF-8 bytes. Encrypted segments are therefore at most 434
  characters.
- The reserved stored name `.hcd-vault.json` can never collide with an encrypted name, because `.`
  is not in the base32 alphabet.
