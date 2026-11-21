# Contract: Storage Provider Interface (internal)

Every storage adapter (`local`, `s3`, `gdrive`) implements this interface. The adapters only ever
see **encrypted names and encrypted bytes**. Encryption happens above this layer, so an adapter
cannot leak plaintext even by mistake (FR-001).

The shared contract test suite `internal/remote/remotetest` encodes every rule below. A new
adapter is accepted only when the full suite passes on Linux, macOS, and Windows (Constitution I
and III).

## Operations

Paths passed to the adapter are lists of already-encrypted segments relative to the remote root.

| Operation | Behavior |
|-----------|----------|
| `Put(ctx, path, r io.Reader, size int64, overwrite bool)` | Streams `r` to `path` and creates missing parent folders. If `overwrite` is false and `path` exists, returns `ErrExists` without writing. Either the whole object becomes visible or nothing does: on any error or context cancel, no object is left at `path`. |
| `Get(ctx, path) (io.ReadCloser, size int64, error)` | Streams the stored bytes. Returns `ErrNotFound` if the object is missing. |
| `Stat(ctx, path) (Entry, error)` | Returns the size, the modification time, and whether the entry is a directory. Returns `ErrNotFound` if missing. |
| `List(ctx, dir) ([]Entry, error)` | Lists the immediate children, both files and folders, in unspecified order. Returns `ErrNotFound` if `dir` is missing. An empty directory gives an empty list. |
| `Delete(ctx, path)` | Removes a file, or an empty folder. Returns `ErrNotFound` if missing. Recursive deletes are composed by the caller. |
| `ReadDescriptor(ctx) / WriteDescriptor(ctx, bytes)` | Reads or writes `.hcd-vault.json` at the root. Writes never overwrite. |

## Errors

Adapters map provider-specific failures to these sentinel errors. The CLI maps them to exit codes,
listed in [cli.md](cli.md).

| Error | Meaning |
|-------|---------|
| `ErrNotFound` | The path does not exist. |
| `ErrExists` | The destination exists and `overwrite` is false. |
| `ErrAuth` | The provider credentials are missing, expired, or revoked. The user needs to reconnect. |
| `ErrQuota` | The provider's storage quota was exceeded. |
| `ErrUnavailable` | A network or transient provider error remained after retries. |

## Behavioral guarantees per adapter

| Guarantee | local | s3 | gdrive |
|-----------|-------|----|--------|
| Atomic put | Temporary file followed by rename | A single PUT, or multipart upload aborted on error | A resumable upload creates the file only when it completes |
| Race-free no-overwrite | `O_EXCL` link and rename check | `If-None-Match: *` | Best effort: check, then upload. Duplicate names are reported by `List` and flagged. |
| Folders | Real directories | Implicit, through key prefixes. `List` uses a delimiter. | Real Drive folders |
| Retries | None | SDK default with backoff | Exponential backoff on 429 and 5xx, up to 5 attempts |
