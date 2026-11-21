# Feature Specification: Encrypted Cloud Storage

**Feature Branch**: `001-encrypted-cloud-storage`

**Created**: 2026-11-20

**Status**: Draft

**Input**: User description: "The purpose of this project is to create, well, essentially, I want to be able to upload things to like Google Drive, or I mean, Amazon, with the encryption key, so that the different cloud providers can't look into whatever cloud provider I have, I can't look into the data that I'm storing there. There should be a symmetric key encryption. So the goal of this program would be to encrypt and decrypt. The files that I'm downloading and uploading."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Securely store and retrieve a file (Priority: P1)

As a person who stores files in the cloud, I want each file encrypted on my own computer with
my secret key before it is uploaded, and decrypted on my computer after it is downloaded, so
that the cloud provider only ever holds data it cannot read.

**Why this priority**: This round trip is the whole reason the product exists. With just this
story and a single provider, the user already gets private cloud storage.

**Independent Test**: Set up a key, connect one provider, upload a known file, inspect the
stored copy through the provider's own web interface (it must be unreadable), then download it
and confirm the result is byte-identical to the original.

**Acceptance Scenarios**:

1. **Given** a configured key and a connected provider, **When** the user uploads a file,
   **Then** the copy stored at the provider contains no readable portion of the original content
   or the original name.
2. **Given** a file previously uploaded with this tool, **When** the user downloads it with the
   same key, **Then** the resulting local file is byte-identical to the original.
3. **Given** a file previously uploaded with this tool, **When** the user tries to download it
   with a different key, **Then** the download is refused with a clear message and no local
   output file is created.
4. **Given** a stored encrypted file that has been altered at the provider, **When** the user
   downloads it, **Then** the tool reports that the file has been tampered with or corrupted and
   does not produce any plaintext output.

---

### User Story 2 - Use more than one cloud provider (Priority: P2)

As a user, I want to store encrypted files on Google Drive and on Amazon using the same key,
so that I can choose where my data lives without changing how I protect it.

**Why this priority**: The user specifically named both Google Drive and Amazon. Supporting a
second provider proves that the protection does not depend on any one provider.

**Independent Test**: Connect both providers, upload the same file to each, download from each,
and confirm both results match the original. Copy an encrypted file by hand from one provider to
the other and confirm it still decrypts.

**Acceptance Scenarios**:

1. **Given** both providers are connected, **When** the user uploads a file and names a provider
   as the destination, **Then** the encrypted file is stored only at that provider.
2. **Given** an encrypted file was moved by hand from one provider to the other, **When** the
   user downloads it from the new provider, **Then** it decrypts correctly.

---

### User Story 3 - See and manage what I have stored (Priority: P2)

As a user, I want to list, download, and delete the encrypted files in my cloud storage using
their original names, so that I can find and manage my files without decrypting everything.

**Why this priority**: Once more than a few files are stored, the user needs to find the right
one. Because names are hidden from the provider (FR-009), this tool is the only way to see what is
stored.

**Independent Test**: Upload several files, list the remote location, check that the original
names are shown, delete one, and check that it is gone at the provider.

**Acceptance Scenarios**:

1. **Given** several files were uploaded, **When** the user lists a remote location, **Then** each
   file is shown with its original name and size.
2. **Given** a stored file, **When** the user deletes it, **Then** it no longer exists at the
   provider and no longer appears in the listing.
3. **Given** a remote location that also contains files not created by this tool, **When** the
   user lists it, **Then** those files are clearly marked as not encrypted by this tool rather
   than causing an error.

---

### User Story 4 - Access my files from another computer (Priority: P3)

As a user with more than one computer (Linux, macOS, or Windows), I want to set up my key on
another machine and download and decrypt my files there.

**Why this priority**: This is valuable, and the constitution requires it (portability), but it
is not needed for the first useful release on a single machine.

**Independent Test**: Upload a file from one operating system, set up the same key on a
different operating system, download the file there, and confirm it matches the original.

**Acceptance Scenarios**:

1. **Given** a file uploaded from one supported operating system, **When** the user sets up the
   same key on a different supported operating system and downloads the file, **Then** it
   decrypts to a byte-identical copy.
2. **Given** a stored file whose original name is not allowed on the downloading platform,
   **When** the user downloads it, **Then** the tool saves it under a safe substitute name and
   tells the user.

---

### Edge Cases

- **Forgotten passphrase or lost key file**: if the user loses the key, the data cannot be recovered by anyone, including the
  tool. The tool must say this clearly during key setup.
- **Interrupted transfer**: a lost network connection, a crash, or a cancel partway through an
  upload or download must not leave a file that looks complete at the provider or on the local
  computer. The user can retry.
- **Very large files**: files larger than the computer's available memory, such as a 10 GB
  video, must still work.
- **Empty files**: zero-byte files can be uploaded and downloaded.
- **Existing local file**: a download that would overwrite an existing local file must not do
  so silently.
- **Existing remote file**: an upload whose destination already holds a file of the same name
  must not overwrite it silently.
- **Provider sign-in problems**: if the sign-in to the provider has expired or been revoked, the
  user is asked to reconnect, and their data and key are not affected.
- **Full storage**: if the provider's storage quota is exceeded, the user gets a clear message
  and no partial file is left behind.
- **Corrupted or tampered files**: these are detected and rejected (see User Story 1,
  scenario 4).
- **Unusual file names**: names with non-English characters, emoji, or very long paths keep
  their original names after a round trip wherever the platform allows it.

## Requirements *(mandatory)*

### Functional Requirements

**Encryption & integrity**

- **FR-001**: The system MUST encrypt file contents on the user's computer before any data is sent
  to a cloud provider. File contents MUST NOT be sent to a provider unencrypted.
- **FR-002**: The system MUST use symmetric authenticated encryption, built on a publicly vetted,
  industry-standard algorithm, so that any change to a stored file is detected when it is
  decrypted.
- **FR-003**: The system MUST decrypt downloaded files so that the result is byte-identical to the
  original file.
- **FR-004**: The system MUST refuse to produce output when decryption fails, whether because of a
  wrong key, corruption, or tampering. It MUST NOT leave a partial or incorrect plaintext file
  behind.
- **FR-005**: The encrypted file format MUST carry a version marker so that later releases can
  still decrypt files written by earlier ones.

**Key management**

- **FR-006**: During key setup, the user MUST be able to choose either (a) a passphrase they
  memorize, from which the key is derived, or (b) a randomly generated key file that they keep
  safe.
- **FR-006a**: A passphrase-derived key MUST be reproducible: entering the same passphrase on any
  supported computer MUST produce the same key, so that files decrypt there. Deriving the key
  MUST be deliberately slow, to resist guessing attacks.
- **FR-006b**: When the user chooses a passphrase, the system MUST reject weak passphrases (for
  example, very short or very common ones) and explain what makes a passphrase acceptable.
- **FR-006c**: A generated key file MUST be produced from a cryptographically secure random
  source. The user MUST be able to copy it to another computer to use it there.
- **FR-007**: The key MUST never be sent to any cloud provider, and MUST NOT be stored in
  plaintext alongside the encrypted data.
- **FR-008**: During key setup, the system MUST warn the user that losing the key means
  permanently losing access to the data, and MUST tell them how to back up the key.

**Metadata privacy**

- **FR-009**: The system MUST encrypt original file and folder names, so that the provider sees
  only random-looking names that reveal nothing about the originals.
- **FR-009a**: Names MUST be decrypted with the user's key for listing and download, so that the
  user always sees and restores the original names and folder structure.

**Cloud providers**

- **FR-010**: The system MUST support Google Drive and Amazon S3 as storage providers.
- **FR-011**: Encrypted files MUST be independent of the provider: a file stored at one provider
  MUST decrypt correctly after being moved to the other provider.
- **FR-012**: The system MUST connect to each provider through that provider's standard
  user-authorization process. Provider credentials MUST be stored securely on the user's
  computer and MUST NOT be stored inside encrypted files.
- **FR-013**: Users MUST be able to choose the destination provider and the folder or location
  within it for each upload.

**User operations**

- **FR-014**: Users MUST be able to upload and download chosen files and folders on demand. An
  automatically synced local folder is out of scope here and planned as a later feature (see
  Assumptions). Nothing in this feature should make adding syncing later harder.
- **FR-015**: Users MUST be able to upload a whole folder, with its relative folder structure
  kept on upload and restored on download.
- **FR-016**: Users MUST be able to list the stored files in a remote location, see original names
  and sizes, and see which files were not created by this tool.
- **FR-017**: Users MUST be able to delete stored files.
- **FR-018**: The system MUST NOT overwrite an existing local or remote file without the user's
  explicit confirmation or an explicit overwrite option.
- **FR-019**: The system MUST show progress for transfers that take longer than a few seconds.
- **FR-020**: The system MUST handle files larger than the computer's available memory.
- **FR-021**: An interrupted upload or download MUST NOT leave a file that looks complete at the
  destination.

**Platform & usability**

- **FR-022**: The system MUST run on Linux, macOS, and Windows, with the same operations and
  behavior on each.
- **FR-023**: A file encrypted on any supported platform MUST decrypt correctly on every other
  supported platform.
- **FR-024**: Error messages MUST state what went wrong and suggest a corrective action.

### Key Entities

- **Encryption Key**: the user's secret, used to encrypt and decrypt contents and names. It is
  either derived from a passphrase or held in a generated key file. It exists only on the
  user's computers, is never shared with providers, and can be backed up by the user.
- **Plain File**: an original, readable file on the user's computer. It has a name, a relative
  path, contents, and a size.
- **Encrypted File**: the protected form of a Plain File as stored at a provider. It holds the
  encrypted contents, the format version, and the information needed to detect tampering. It is
  stored under an encrypted name, from which the original name can be recovered only with the
  key.
- **Provider Connection**: the user's authorized link to one cloud provider account. It holds
  the provider type, the authorization (stored securely on the computer), and a default
  location.
- **Remote Location**: a folder or bucket within a provider where Encrypted Files are kept.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: When stored files are inspected through the provider's own web interface or
  download, 0% of the original file contents, file names, and folder names can be read.
- **SC-002**: 100% of round trips produce byte-identical files, for test files from 0 bytes up to
  10 GB.
- **SC-003**: 100% of stored files that have a single byte changed are detected and rejected on
  download.
- **SC-004**: 100% of attempts to decrypt with the wrong key are rejected, and no output file is
  created.
- **SC-005**: A first-time user can set up a key, connect a provider, and upload their first file
  in under 10 minutes by following the documentation.
- **SC-006**: Encryption and decryption add no more than 10% to the total transfer time, compared
  with an unencrypted transfer of the same file over the same connection.
- **SC-007**: A file uploaded from each supported operating system downloads and decrypts
  correctly on each of the other two.

## Assumptions

- The tool is for a single person protecting their own files. Sharing encrypted files with other
  people is out of scope for this version.
- "Amazon" means Amazon S3 storage. Amazon Drive was discontinued in 2023, and Amazon Photos is
  not a general file store.
- The user already has a Google Drive and/or Amazon S3 account and can sign in to it.
- Losing the key is unrecoverable by design. There is no "forgot password" recovery, because any
  recovery path would weaken the protection.
- Providers can still see some information about stored files: approximate file sizes, the
  number of files, upload and download times, and access patterns. Hiding this information is
  out of scope.
- This version targets desktop operating systems (Linux, macOS, Windows). Mobile apps are out of
  scope.
- An automatically synced local folder is deferred to a later feature. This version supports
  on-demand upload and download only.
- Keeping file versions, and resolving conflicting edits made from several computers at once,
  are out of scope for this version.
- Because names are encrypted, the provider's own website and apps cannot be used to find files.
  Users find their files through this tool's listing.
- An internet connection is needed for uploading, downloading, listing, and deleting.
