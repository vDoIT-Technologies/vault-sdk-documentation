# Bot Operations

This section covers creating bots, uploading files to their dedicated folders,
and adding existing drive files and folders to bot knowledge.

## Create Bot

### `createBot(vaultId, bot)`

Creates a bot for the specified vault and automatically creates its dedicated
folder. The vault must be associated with your configured client API key and
have SDK access enabled. See **[User Operations](../User%20Operations/README.md)**
for creating or importing a vault.

**Parameters:**

- `vaultId` (String): The ID of the vault that will own the bot. Required.
- `bot` (Object): The bot configuration. Required.
- `bot.name` (String): The bot's display name. Required, non-empty, and at most
  50 characters.
- `bot.description` (String, optional): The bot's personality or description.
  At most 100 characters.
- `bot.profession` (String, optional): The bot's profession label. At most
  50 characters. Accepts letters, numbers, spaces, and `. , ' - & ( ) /`.

The SDK trims the supplied text before sending it. Length limits and the
profession character restrictions are enforced by the backend.

**Example:**

```javascript
try {
  const result = await vault.createBot('your-vault-id', {
    name: 'Support Bot',
    description: 'Answers customer questions clearly',
    profession: 'Customer Support',
  });

  console.log('Bot created:', result.data);
  console.log('Bot ID:', result.data.id);
  console.log('Bot folder:', result.data.folder);
} catch (error) {
  console.error('Error creating bot:', error.code, error.message);
}
```

Only the name is required in the bot configuration:

```javascript
const result = await vault.createBot('your-vault-id', {
  name: 'My Bot',
});
```

**Response:**

Returns an object with `success`, `message`, and `data`:

- `success` (Boolean): `true` when the bot is created.
- `message` (String): `"Bot created successfully"`.
- `data` (Object): The created bot's details, including its `id` and dedicated
  `folder` object.

**Errors:**

- `INVALID_PARAMETER`: A required parameter is missing or a parameter fails SDK
  validation.
- `BAD_REQUEST`: The backend rejects the bot fields, such as an exceeded length
  limit or unsupported characters in the profession.
- `UNAUTHORIZED`: Authentication fails or the vault is not associated with the
  configured client API key.
- `FORBIDDEN`: SDK access is disabled for the vault user.
- `NOT_FOUND`: The vault user does not exist.

See **[Error Handling](../Error%20Handling/README.md)** for error types and other
API or network errors.

## Upload Files to a Bot

### `uploadFilesToBot(files, vaultId, botId)`

Uploads one or more files to the bot's dedicated folder and starts bot knowledge
processing in the background. The SDK reads each file, sanitizes its name,
resolves its MIME type, hashes its content, obtains a presigned storage URL,
uploads the bytes, and registers the file for ingestion. Files upload in parallel.
A successful upload does not mean knowledge processing has finished.

**Parameters:**

- `files` (String | Object | File | Blob | Array): A single file or a non-empty
  array of files. Required. Each file can be a path string, a named bytes object
  such as `{ buffer: fileBuffer, name: 'faq.pdf' }`, a path object such as
  `{ path: './faq.pdf' }`, or a `File` / `Blob`. See
  [Accepted file forms](../File%20Operations/README.md#accepted-file-forms)
  for supported fields and MIME type handling.
- `vaultId` (String): The ID of the vault that owns the bot. Required.
- `botId` (String): The target bot's ID, returned as `data.id` by `createBot()`.
  Required. The SDK uses the bot's dedicated folder automatically.

For an object input, `durationSeconds` may also be supplied as a finite,
non-negative number to include the media duration during registration.

**Single-file example:**

```javascript
try {
  const result = await vault.uploadFilesToBot(
    './faq.pdf',
    'your-vault-id',
    'bot-id'
  );

  console.log(result.message);
  console.log('Files sent for ingestion:', result.data.files);
  console.log('Skipped:', result.data.skipped);
} catch (error) {
  console.error('Bot upload failed:', error.code, error.message);
  console.error('File results:', error.data?.results);
}
```

**Multiple-file example:**

```javascript
try {
  const result = await vault.uploadFilesToBot(
    ['./faq.pdf', { path: './product-guide.txt' }],
    'your-vault-id',
    'bot-id'
  );

  for (const entry of result.data.results) {
    if (entry.status === 'success') {
      console.log('Uploaded:', entry.fileName, entry.response);
    } else {
      console.error('Failed:', entry.fileName, entry.code, entry.error);
    }
  }
  console.log('Skipped:', result.data.skipped);
} catch (error) {
  console.error('Bot upload failed:', error.code, error.message);
  for (const entry of error.data?.results ?? []) {
    console.error('Failed:', entry.fileName, entry.code, entry.error);
  }
}
```

**Response:**

When at least one file upload succeeds, returns an object with:

- `success` (Boolean): `true`, including when some files fail.
- `message` (String): A summary of files sent for ingestion and skipped entries.
- `data.files` (Array): File details collected from successful registration
  responses.
- `data.skipped` (Array): Backend-reported skipped entries, plus failed uploads
  represented as `{ name, reason, code }`.
- `data.results` (Array): One result per input file, in input order. Successful
  entries contain `{ status: 'success', fileName, response }`, where `response`
  is the registration response. Failed entries contain
  `{ status: 'failed', fileName, error, code }`.
- `data.successCount` (Number): Number of successful upload entries.
- `data.failureCount` (Number): Number of failed upload entries.

`successCount` counts successful calls, including those whose registration
response reports skipped entries; it does not count completed knowledge
processing. The `fileName` in each result is an input label. Read the stored
file details from the registration response for the sanitized name.

**Limits and errors:**

Each file must contain readable, non-empty content and must not exceed the SDK's
10 GB limit. A bare `Buffer` is not accepted; wrap it with a file name.

- Missing or invalid `vaultId` or `botId` throws a `ValidationError` with code
  `INVALID_PARAMETER` before any uploads start.
- An empty `files` array throws a `VaultError` with code `INVALID_PARAMETER`.
- If some files fail, the method resolves with their errors in `data.results`
  and `data.skipped`.
- If every file fails, including a failed single-file upload, the method throws
  a `VaultError` with code `UPLOAD_FAILED`. Inspect `error.data.results` for
  each file's underlying error.

Per-file error codes can include `FILE_READ_FAILED`, `INVALID_PARAMETER`,
`FILE_TOO_LARGE`, `PRESIGN_FAILED`, `STORAGE_UPLOAD_FAILED`, and `REGISTER_FAILED`.
API and network errors retain their SDK error codes. See
**[Error Handling](../Error%20Handling/README.md)** for their meanings.

## Add Drive Files to a Bot

### `addDriveFilesToBot(vaultId, botId, fileIds)`

Links existing files in the vault's drive to a bot's knowledge set. Files remain
in their current storage location and do not need to be uploaded again.

**Parameters:**

- `vaultId` (String): The ID of the vault that owns the bot and drive files.
  Required.
- `botId` (String): The target bot's ID. Required.
- `fileIds` (String | Array<String>): One existing drive file ID or an array of
  file IDs. Required. Use file IDs from `getFiles()` or `getAllFiles()`; see
  [Retrieve Files](../File%20Operations/README.md#retrieve-files).

The SDK trims file IDs, removes duplicates, and discards blank or non-string
array entries. At least one ID must remain. The backend accepts up to 50 unique
file IDs per request. Each ID must identify a file owned by the vault user.

**Single-file example:**

```javascript
try {
  const result = await vault.addDriveFilesToBot(
    'your-vault-id',
    'bot-id',
    'file-id'
  );

  console.log(result.message);
  console.log(result.data.results);
} catch (error) {
  console.error('Could not add drive file:', error.code, error.message);
  console.error('File results:', error.data?.data?.results);
}
```

**Multiple-file example:**

```javascript
try {
  const result = await vault.addDriveFilesToBot(
    'your-vault-id',
    'bot-id',
    ['file-id-1', 'file-id-2']
  );

  for (const entry of result.data.results) {
    if (entry.success) {
      console.log(entry.alreadyLinked ? 'Already linked:' : 'Added:', entry.fileId);
    } else {
      console.error('Failed:', entry.fileId, entry.error);
    }
  }
} catch (error) {
  console.error('Could not add drive files:', error.code, error.message);
  for (const entry of error.data?.data?.results ?? []) {
    console.error('Failed:', entry.fileId, entry.error);
  }
}
```

**Response:**

Returns the backend response with:

- `success` (Boolean): `true` for a successful request, including partial failures.
- `message` (String): A summary of files added, already linked, and failed.
- `data.results` (Array): Per-file outcomes containing `fileId`, `success`,
  `fileName` when available, `alreadyLinked: true` for files already in the bot,
  and `error` for failed entries. Match entries by `fileId`; their order may
  differ from the input order.
- `data.successCount` (Number): Successful entries, including already-linked files.
- `data.failureCount` (Number): Failed entries.

Files already linked to the bot count as successes and are not added again.
Check each entry's `success` boolean to identify partial failures.

**Limits and errors:**

The backend accepts PDF, TXT, and DOCX documents up to 10 MB; MP3, WAV, and OGG
audio up to 200 MB; and MP4, MOV, and MKV video up to 3 GB.

- Missing or invalid `vaultId` or `botId` throws a `ValidationError` with code
  `INVALID_PARAMETER`.
- If no usable file IDs remain after normalization, the SDK throws a `VaultError`
  with code `INVALID_PARAMETER`.
- More than 50 unique file IDs causes a `BAD_REQUEST` error.
- Missing files, unsupported files, unreadable storage content, and ingestion
  failures are reported per file in `data.results` when other entries succeed.
- If every file fails, the SDK throws a `VaultError` with code `BAD_REQUEST`.
  The full backend response is in `error.data`, so per-file details are in
  `error.data.data.results`.

Authentication, ownership, API, and network failures may also reject the request.
See **[Error Handling](../Error%20Handling/README.md)** for SDK error types and codes.

## Add Drive Folders to a Bot

### `addDriveFoldersToBot(vaultId, botId, folderIds)`

Attaches existing drive folders to a bot and ingests compatible files from those
folders, including nested folders. The folders remain in their current storage
location. Unsupported or unreadable files are skipped and reported in the result.

**Parameters:**

- `vaultId` (String): The ID of the vault that owns the bot and folders. Required.
- `botId` (String): The target bot's ID. Required.
- `folderIds` (String | Array<String>): One existing drive folder ID or an array
  of folder IDs. Required. Each ID must identify a folder owned by the vault user.

The SDK trims folder IDs, removes duplicates, and discards blank or non-string
array entries. At least one ID must remain. The backend accepts up to 50 unique
folder IDs per request.

**Single-folder example:**

```javascript
try {
  const result = await vault.addDriveFoldersToBot(
    'your-vault-id',
    'bot-id',
    'folder-id'
  );

  console.log(result.message);
  console.log('Folder and ingestion results:', result.data.results);
} catch (error) {
  console.error('Could not add folder:', error.code, error.message);
  console.error('Folder results:', error.data?.data?.results);
}
```

**Multiple-folder example:**

```javascript
try {
  const result = await vault.addDriveFoldersToBot(
    'your-vault-id',
    'bot-id',
    ['folder-id-1', 'folder-id-2']
  );

  for (const entry of result.data.results) {
    if (!entry.success) {
      console.error('Folder failed:', entry.folderId, entry.error);
      continue;
    }

    console.log(entry.alreadyLinked ? 'Already attached:' : 'Attached:', entry.folderId);
    console.log('Files ingested:', entry.linkedFiles);
    console.log('Skipped files:', entry.brainIngestion.skipped);
    if (entry.brainIngestion.error) {
      console.error('Ingestion failed:', entry.brainIngestion.error);
    }
  }
} catch (error) {
  console.error('Could not add folders:', error.code, error.message);
  for (const entry of error.data?.data?.results ?? []) {
    console.error('Folder failed:', entry.folderId, entry.error);
  }
}
```

**Response:**

Returns the backend response with:

- `success` (Boolean): `true` for a successful request, including partial failures.
- `message` (String): A summary of folders added or already attached, files
  ingested, and failed folders.
- `data.results` (Array): Per-folder outcomes in normalized input order.
- `data.successCount` (Number): Successful folder entries, including folders
  already attached to the bot.
- `data.failureCount` (Number): Failed folder entries.

Successful entries in `data.results` contain:

- `folderId` and `folderName`: The folder's ID and name.
- `success`: `true`.
- `alreadyLinked` (Boolean): Whether the folder was already attached.
- `linkedFiles` (Number): The number of files ingested during this call.
- `brainIngestion` (Object): Contains `ingested` (the same count as `linkedFiles`),
  `skipped` (an array of skipped-file descriptions), and `error` when ingestion
  failed.

Failed entries contain `{ folderId, success: false, error }`.

A successful folder entry does not guarantee that any files were ingested. Empty
folders, folders containing only skipped files, and folders whose ingestion fails
can still have `success: true`. Check `linkedFiles`, `brainIngestion.skipped`,
and `brainIngestion.error`. Calling the method for an already-attached folder
still attempts ingestion of its contents.

**Limits and errors:**

Files within folders use the same supported types and size limits as
[`addDriveFilesToBot()`](#add-drive-files-to-a-bot): PDF, TXT, and DOCX documents
up to 10 MB; MP3, WAV, and OGG audio up to 200 MB; and MP4, MOV, and MKV video
up to 3 GB.

- Missing or invalid `vaultId` or `botId` throws a `ValidationError` with code
  `INVALID_PARAMETER`.
- If no usable folder IDs remain after normalization, the SDK throws a
  `VaultError` with code `INVALID_PARAMETER`.
- More than 50 unique folder IDs causes a `BAD_REQUEST` error.
- Individual folder failures are returned in `data.results` when other folders
  succeed.
- If every folder entry fails, the SDK throws a `VaultError` with code
  `BAD_REQUEST`. Inspect `error.data.data.results` for per-folder errors.
  Ingestion errors inside successful folder entries do not trigger this exception.

Authentication, ownership, API, and network failures may also reject the request.
See **[Error Handling](../Error%20Handling/README.md)** for SDK error types and codes.
