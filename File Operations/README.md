# File Operations

This section covers all operations related to file management within the Vault.

## Upload Files

Uploading is a single call. Hand the SDK a file and it does the rest: reads the
bytes, sanitizes the name, resolves the MIME type, hashes the content, requests a
presigned storage URL, uploads the bytes, and registers the file with the Vault
backend.

> **Note:** Earlier versions exposed `getPresignedUrl()` and `registerUpload()` as
> separate methods for a manual three-step upload flow. Those are no longer part of
> the public API — `uploadFile()` and `uploadFiles()` perform all three steps
> internally.

### `uploadFile(file, vaultId, parentId)`

Uploads a single file.

**Parameters:**

- `file` (String | Object | Blob): The file to upload. See [Accepted file forms](#accepted-file-forms) below.
- `vaultId` (String): The ID of the vault where the file will be uploaded.
- `parentId` (String, optional): The ID of the parent folder. Defaults to `null` (root).

**Example:**

```javascript
// Straight from disk — name and MIME type are derived from the file itself
const result = await vault.uploadFile('./photo.jpg', 'your-vault-id');

// Upload into a specific folder
const inFolder = await vault.uploadFile(
  './report.pdf',
  'your-vault-id',
  'parent-folder-id'
);

console.log(result);
```

#### Accepted file forms

`file` may be any of the following:

| Form | Example |
| --- | --- |
| Path on disk | `'./photo.jpg'` |
| Bytes + name | `{ buffer: fileBuffer, name: 'report.pdf' }` |
| Path in an object | `{ path: './photo.jpg' }` |
| `File` / `Blob` | `new File([bytes], 'photo.jpg', { type: 'image/jpeg' })` |

Notes on the object form:

- The bytes may be supplied as `buffer`, `data`, `content`, or `bytes`, and may be
  a `Buffer`, `ArrayBuffer`, or any typed array (e.g. `Uint8Array`).
- The name may be supplied as `name`, `fileName`, or `filename`; if only `path` is
  given, the name is taken from the last path segment.
- `type` (or `mimeType` / `contentType`) is **optional** — it is derived from the
  file extension when omitted, falling back to `application/octet-stream`.
- A bare `Buffer` is **not** accepted, because it carries no file name. Pass
  `{ buffer, name }` instead.

#### File name sanitization

Names are sanitized to ASCII before upload, mirroring the sanitizer the backend
would apply itself. Accents are folded, non-ASCII characters and the illegal
characters `< > : " / \ | ? *` are stripped, whitespace is collapsed, and leading
or trailing dots and spaces are removed. Windows reserved names (`CON`, `PRN`,
`AUX`, `NUL`, `COM1`–`COM9`, `LPT1`–`LPT9`) are prefixed with an underscore, and
the final name is capped at 255 characters.

```
héllo wörld🤣.PNG   →   hello world.PNG
```

The stored name is returned in the upload response, so read it from there rather
than assuming the name you passed in.

#### Limits and failures

`uploadFile()` throws a `VaultError` when:

| Condition | `error.code` |
| --- | --- |
| The path or object could not be read | `FILE_READ_FAILED` |
| The file is missing, malformed, or zero bytes | `INVALID_PARAMETER` |
| The file exceeds the 10 GB maximum | `FILE_TOO_LARGE` |
| The presigned URL could not be obtained | `PRESIGN_FAILED` |
| The bytes failed to upload to storage | `STORAGE_UPLOAD_FAILED` |
| The upload succeeded but registration failed | `REGISTER_FAILED` |

A missing or invalid `vaultId` throws a `ValidationError` instead. See
**[Error Handling](../Error%20Handling/README.md)** for the full list.

### `uploadFiles(files, vaultId, parentId)`

Uploads multiple files in parallel. Each file is uploaded independently, so one
failure does not block the others — the promise resolves with a result for every
entry rather than rejecting.

**Parameters:**

- `files` (Array): A non-empty array of files, each in any form `uploadFile()` accepts.
- `vaultId` (String): The ID of the vault where files will be uploaded.
- `parentId` (String, optional): The ID of the parent folder. Defaults to `null` (root).

**Example:**

```javascript
const results = await vault.uploadFiles(
  [
    './file1.pdf',
    { buffer: fs.readFileSync('path/to/file2.jpg'), name: 'file2.jpg' },
    { path: './file3.txt' },
  ],
  'your-vault-id'
);

for (const result of results) {
  if (result.status === 'success') {
    console.log('Uploaded:', result.fileName);
  } else {
    console.error('Failed:', result.fileName, result.code, result.error);
  }
}
```

**Response:**

An array in the same order as `files`, where each entry carries a `status`:

```javascript
// Success — the registration response, plus:
{ status: 'success', fileName: 'file1.pdf', /* ...registration fields */ }

// Failure — the error is reported, not thrown:
{ status: 'failed', fileName: 'file2.jpg', error: '...', code: 'STORAGE_UPLOAD_FAILED' }
```

---

## Retrieve Files

### `getFiles(vaultId, query)`

Searches for files by name.

**Parameters:**

- `vaultId` (String): The ID of the vault.
- `query` (String, optional): Search query string. Defaults to `""` (no filter).

**Example:**

```javascript
const files = await vault.getFiles('your-vault-id', 'search-term');
console.log(files);
```

### `getAllFiles(vaultId)`

Retrieves all files in the vault.

**Parameters:**

- `vaultId` (String): The ID of the vault.

**Example:**

```javascript
const allFiles = await vault.getAllFiles('your-vault-id');
console.log(allFiles);
```

## Manage Files

### `deleteFile(vaultId, fileId)`

Deletes a specific file from the vault.

**Parameters:**

- `vaultId` (String): The ID of the vault.
- `fileId` (String): The unique ID of the file to delete.

**Example:**

```javascript
const response = await vault.deleteFile('your-vault-id', 'file-id-to-delete');
console.log(response);
```

### `renameItem(vaultId, itemId, newName)`

Renames a file or a folder.

**Parameters:**

- `vaultId` (String): The ID of the vault.
- `itemId` (String): The unique ID of the file or folder.
- `newName` (String): The new name for the item.

**Example:**

```javascript
const response = await vault.renameItem('your-vault-id', 'item-id', 'New Name.txt');
console.log(response);
```

> **`renameFile(vaultId, itemId, newName)`** is kept as a backward-compatible alias
> for `renameItem()` and behaves identically. Prefer `renameItem()` in new code —
> it works for folders as well as files.

## Starred Files

### `addToStarred(vaultId, fileId, isStarred)`

Marks or unmarks a file as starred (favorite).

**Parameters:**

- `vaultId` (String): The ID of the vault.
- `fileId` (String): The unique ID of the file.
- `isStarred` (Boolean): `true` to star, `false` to unstar. Must be a boolean.

**Example:**

```javascript
// Add to starred
await vault.addToStarred('your-vault-id', 'file-id', true);

// Remove from starred
await vault.addToStarred('your-vault-id', 'file-id', false);
```

### `getStarredFiles(vaultId)`

Retrieves all files marked as starred.

**Parameters:**

- `vaultId` (String): The ID of the vault.

**Example:**

```javascript
const starredFiles = await vault.getStarredFiles('your-vault-id');
console.log(starredFiles);
```
