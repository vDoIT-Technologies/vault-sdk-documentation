# Bot Operations

This section covers creating bots, retrieving their details, uploading files to their dedicated folders,
and adding existing drive files and folders to bot knowledge. For live chat, see
**[Bot Chat Operations](../Bot%20Chat%20Operations/README.md)**.
For retrieving saved chat sessions and message history, see
**[Bot Session Operations](../Bot%20Session%20Operations/README.md)**.

## Quote Transcription

### `quoteTranscription(vaultId, botId, payload)`

Estimates the Twin Points cost of transcribing audio and video before you upload
files or attach drive folders to a bot. The quote does not reserve or charge
points and does not start ingestion.

**Parameters:**

- `vaultId` (String): The vault that owns the target bot. Required.
- `botId` (String): The target bot ID. Required.
- `payload` (Object, optional): Inputs to quote. Defaults to `{}` and must
  contain at least one usable file or folder ID.
  - `files` (Array<Object>, optional): Direct file metadata. Each entry supports:
    - `name` (String): File name, required for that entry.
    - `size` (Number): File size in bytes. Defaults to `0` when omitted.
    - `fileId` (String, optional): Existing drive asset ID when quoting a linked
      file. Existing transcripts can reduce its estimate to zero.
    - `durationSeconds` (Number, optional): Measured media duration. When
      supplied, it is preferred over estimating duration from file size.
  - `folderIds` (Array<String>, optional): Existing drive folder IDs to inspect.
    Folder contents, including nested folders, are scanned for media files.

The SDK trims file names and IDs, removes duplicate folder IDs, ignores malformed
file entries and blank IDs, and includes only supported audio/video extensions in
the estimate. Non-media files are ignored. Up to 50 folders may be quoted in one
request.

**Quote direct files:**

```javascript
try {
  const result = await vault.quoteTranscription('your-vault-id', 'bot-id', {
    files: [
      { name: 'customer-call.mp3', size: 12_000_000, durationSeconds: 905 },
      { name: 'demo.mp4', size: 80_000_000 },
    ],
  });

  console.log('Estimated points:', result.data.estimatedPoints);
  console.log('Available points:', result.data.availablePoints);
  console.log('Enough points:', result.data.enough);
} catch (error) {
  console.error('Could not quote transcription:', error.code, error.message);
}
```

**Quote drive files and folders:**

```javascript
const result = await vault.quoteTranscription('your-vault-id', 'bot-id', {
  files: [
    { name: 'podcast.mp3', size: 25_000_000, fileId: 'drive-file-id' },
  ],
  folderIds: ['folder-id-1', 'folder-id-2'],
});
```

Use the same payload before `uploadFilesToBot()`, `addDriveFilesToBot()`, or
`addDriveFoldersToBot()` when you want to show a user the expected transcription
cost. A file listed directly and discovered through a folder is counted once.

**Response:**

Returns the standard response:

```javascript
{
  success: true,
  message: 'Transcription quote',
  data: {
    files: [
      {
        name: 'customer-call.mp3',
        isVideo: false,
        alreadyTranscribed: false,
        estimatedPoints: 7
      }
    ],
    estimatedPoints: 7,
    availablePoints: 120,
    enough: true
  }
}
```

- `data.files` contains one entry for each unique supported media candidate;
  documents and unsupported file types are omitted.
- `alreadyTranscribed` is `true` when the backend finds existing extracted
  transcript content; that item has `estimatedPoints: 0`.
- `estimatedPoints` is the total estimate for all returned media entries.
- `availablePoints` is the user's current available Twin Points balance.
- `enough` is `true` when `availablePoints >= estimatedPoints`.

Estimates use measured duration when available, otherwise file size and media
type. The final charge can differ because the actual transcription service usage
is settled after processing.

**Errors:**

- `INVALID_PARAMETER`: `vaultId` or `botId` is invalid, `payload` is not an
  object, or no usable file entries or folder IDs were supplied.
- `BAD_REQUEST`: More than 50 folder IDs were supplied, a folder cannot be
  inspected, or the backend rejects the quote request.
- `NOT_FOUND`: The bot, folder, or referenced drive file cannot be found or is
  not owned by the authenticated user.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.

See **[Error Handling](../Error%20Handling/README.md)** for SDK error types and
other API or network errors.

## Update Bot

### `updateBot(vaultId, botId, updates)`

Updates one or more editable bot settings. This is a partial update: omit fields
that should remain unchanged. Changes to `description` and `profession` are also
sent to the bot's personality service so the bot's prompt stays synchronized.

**Parameters:**

- `vaultId` (String): The vault that owns the bot. Required.
- `botId` (String): The bot ID to update. Required.
- `updates` (Object): Fields to change. Required. Supported fields are:
  - `name` (String): Display name. Must be non-empty and at most 50 characters.
  - `description` (String): Bot personality or description. At most 100 characters.
  - `profession` (String): Profession label. At most 50 characters; accepts
    letters, numbers, spaces, and `. , ' - & ( ) /`.
  - `useLLMFallback` (Boolean): Whether the bot may fall back to the configured
    LLM when its primary response service is unavailable.
  - `wordLimit` (Integer): Response word limit from 10 through 800.

The SDK trims `name` before sending it. The backend trims text fields as it
saves them. Unsupported fields are not included in the request. Passing an
explicit value of the wrong type throws `INVALID_PARAMETER` before the request
is sent.

**Example:**

```javascript
try {
  const result = await vault.updateBot('your-vault-id', 'bot-id', {
    name: 'Support Assistant',
    description: 'Answers customer questions clearly and concisely',
    profession: 'Customer Support',
    useLLMFallback: true,
    wordLimit: 300,
  });

  console.log(result.message); // Bot updated successfully
  console.log('Updated bot:', result.data);
} catch (error) {
  console.error('Could not update bot:', error.code, error.message);
}
```

A single-field update is also valid:

```javascript
await vault.updateBot('your-vault-id', 'bot-id', { wordLimit: 200 });
```

**Response:**

Returns the standard response:

```javascript
{
  success: true,
  message: 'Bot updated successfully',
  data: {
    id: 'bot-id',
    name: 'Support Assistant',
    description: 'Answers customer questions clearly and concisely',
    profession: 'Customer Support',
    wordLimit: 300,
    useLLMFallback: true,
    llm: null
  }
}
```

`data` is the updated serialized bot. Sensitive custom LLM credentials are not
returned; any configured `llm` object contains metadata and an API-key hint only.

**Errors:**

- `INVALID_PARAMETER`: A required argument has the wrong type, the name is
  empty, or a field fails SDK validation.
- `BAD_REQUEST`: A name exceeds 50 characters, a description exceeds 100, a
  profession exceeds 50 or contains unsupported characters, `wordLimit` is
  outside 10–800 or is not an integer, or `useLLMFallback` is not boolean.
- `NOT_FOUND`: The bot does not exist or is not owned by the vault user.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.
- `CONFLICT`: The backend reports a conflicting bot update.

See **[Error Handling](../Error%20Handling/README.md)** for SDK error types and
other API or network errors.

## Remove a Bot Asset

### `removeBotAsset(vaultId, botId, assetType, assetId, options)`

Removes a file or a linked drive folder from a bot. The behavior depends on
`assetType`: removing a folder only unlinks it, while removing a file can either
detach it from the bot or permanently delete its drive copy.

**Parameters:**

- `vaultId` (String): The vault that owns the bot. Required.
- `botId` (String): The bot ID. Required.
- `assetType` (String): Must be `'file'` or `'folder'`. The SDK trims and
  lowercases this value.
- `assetId` (String): The bot file ID when `assetType` is `'file'`, or the drive
  folder ID when `assetType` is `'folder'`. Required.
- `options` (Object, optional): File-removal flags. Defaults to `{}`.
  `permanent` and `keepTranscript` are supported only for files.

| Option | Type | Applies to | Behavior |
| --- | --- | --- | --- |
| `permanent` | Boolean | Files | `false` (default) detaches the file and preserves its drive copy. `true` also deletes the drive file and its stored transcript. |
| `keepTranscript` | Boolean | Files | With `permanent: true`, deletes the original media while keeping its extracted transcript as a drive file. Requires a stored transcript. |

Folder removal supports neither option: supplying `permanent` or
`keepTranscript` for `assetType: 'folder'` throws `INVALID_PARAMETER`.

**Remove a file from the bot:**

```javascript
try {
  const result = await vault.removeBotAsset(
    'your-vault-id',
    'bot-id',
    'file',
    'bot-file-id'
  );

  console.log(result.message);
  console.log(result.data);
} catch (error) {
  console.error('Could not remove bot file:', error.code, error.message);
}
```

With the default `permanent: false`, bot-uploaded files are moved to the drive
root when possible. Files originally linked from elsewhere stay in their
existing drive location.

**Permanently remove a file but keep its transcript:**

```javascript
const result = await vault.removeBotAsset(
  'your-vault-id',
  'bot-id',
  'file',
  'bot-file-id',
  { permanent: true, keepTranscript: true }
);
console.log(result.data.transcriptKept); // true
```

**Unlink a folder:**

```javascript
const result = await vault.removeBotAsset(
  'your-vault-id',
  'bot-id',
  'folder',
  'folder-id'
);
console.log(result.message); // Folder removed from the bot. It is still in your drive.
```

Unlinking a folder does not delete the folder or its files. It also does not
remove the bot's dedicated folder.

**Response:**

Returns the standard `{ success, message, data }` response. The message and
data fields vary by action:

- File detach: `data.botFileRemoved` is `true`, `data.driveFileRemoved` is
  `false`, and `data.movedToRoot` indicates whether a bot-uploaded file moved to
  the drive root.
- Permanent file delete: `data.botFileRemoved` and `data.driveFileRemoved` are
  `true`; `data.storageFreed` reports released storage.
- Permanent media delete with transcript retention: `data.transcriptKept` is
  `true`, `data.driveFileRemoved` is `true`, and `data.transcriptFile` identifies
  the retained transcript when available.
- Folder unlink: `data.id`, `data.name`, `data.parentId`, `data.botId`, and
  `data.removedFiles` describe the unlinked folder.

**Errors:**

- `INVALID_PARAMETER`: A required ID is invalid, `assetType` is not `'file'` or
  `'folder'`, file-only options are used for a folder, or an option is not a
  boolean.
- `BAD_REQUEST`: A transcript was requested but none exists, the file is shared
  with another bot and cannot be permanently deleted, or the asset operation is
  not allowed.
- `NOT_FOUND`: The bot or asset does not exist, is not owned by the user, or the
  file/folder is not associated with the bot.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.

Permanent deletion is irreversible. Use the default detach behavior when the
drive copy should remain available.

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

## Delete Bot

### `deleteBot(botId, vaultId)`

Deletes a bot owned by the authenticated vault user and removes its bot brain.
The bot's linked drive folders and files are preserved in the vault; deleting a
bot does not delete those drive assets or release their storage.

> Note the argument order: this method takes `botId` first and `vaultId` second,
> unlike most bot methods in this SDK.

**Parameters:**

- `botId` (String): The ID of the bot to delete. Required.
- `vaultId` (String): The ID of the vault that owns the bot. Required.

**Example:**

```javascript
try {
  const result = await vault.deleteBot('bot-id', 'your-vault-id');
  console.log(result.message); // Bot deleted successfully...
} catch (error) {
  console.error('Could not delete bot:', error.code, error.message);
}
```

The deletion is permanent for the bot and its associated knowledge. Confirm
that the bot ID is correct before calling this method. Files and folders linked
to the bot remain available in the drive and can be linked to another bot later.

**Response:**

Returns the standard response:

```javascript
{
  success: true,
  message: 'Bot deleted successfully. Linked folder has been unlinked and preserved.',
  data: null
}
```

**Errors:**

- `INVALID_PARAMETER`: `botId` or `vaultId` is missing or fails SDK validation.
- `NOT_FOUND`: The bot does not exist or is not owned by the vault user.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.
- `SERVER_ERROR`: The bot or its brain could not be deleted because of a backend
  or external service failure.

See **[Error Handling](../Error%20Handling/README.md)** for SDK error types and
other API or network errors.

## Get Bot Details

### `getBotDetails(vaultId, botId)`

Retrieves one bot's details, including its associated files and linked drive
folders. Omit `botId` to retrieve details for every bot owned by the vault user.

**Parameters:**

- `vaultId` (String): The ID of the vault that owns the bot or bots. Required.
- `botId` (String, optional): The ID of a specific bot. When omitted, returns
  all bots in the vault.

The first argument is always `vaultId`. To retrieve a specific bot, pass both
arguments; `getBotDetails(botId)` would treat that ID as a vault ID.

**Single-bot example:**

```javascript
try {
  const result = await vault.getBotDetails('your-vault-id', 'bot-id');

  console.log('Bot:', result.data);
  console.log('Files:', result.data.files);
  console.log('Linked folders:', result.data.folders);
} catch (error) {
  console.error('Could not fetch bot:', error.code, error.message);
}
```

**All-bots example:**

```javascript
try {
  const result = await vault.getBotDetails('your-vault-id');

  for (const bot of result.data) {
    console.log('Bot:', bot.id, bot.name);
    console.log('Files:', bot.files);
    console.log('Linked folders:', bot.folders);
  }
} catch (error) {
  console.error('Could not fetch bots:', error.code, error.message);
}
```

**Response:**

Returns an object with:

- `success` (Boolean): `true` when the request succeeds.
- `message` (String): `"Bot fetched successfully"` for one bot or
  `"Bots fetched successfully"` for all bots.
- `data` (Object | Array): A detailed bot object when `botId` is supplied,
  or an array of detailed bot objects when it is omitted. The array is empty
  when the vault user has no bots.

Each bot includes its metadata, such as `id`, `name`, `description`, and
`profession`, plus:

- `files` (Array): Associated bot files, newest first, including file details,
  processing status, and drive asset information. Entries also include
  `sharedWith` and `canDeletePermanently` to describe sharing with other bots.
- `folders` (Array): Linked drive folders, including their `folderId`, `name`,
  and `parentId`. These are folder links, not the bot's dedicated folder object
  returned by `createBot()`.
- `llm` (Object | null): Custom model configuration when configured; otherwise
  `null`. Includes provider and model details and an API key hint, without
  returning the stored API key.

**Errors:**

- `INVALID_PARAMETER`: `vaultId` is missing or invalid, or a supplied `botId`
  fails SDK validation.
- `NOT_FOUND`: The requested bot does not exist or is not owned by the vault user.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.

See **[Error Handling](../Error%20Handling/README.md)** for SDK error types and
other API or network errors.

## Get Bot File Text

### `getBotFileText(vaultId, botId, fileId)`

Retrieves the extracted text for a file associated with a bot. The backend
returns cached text when available. Otherwise, it attempts extraction from a
stored TXT, PDF, or DOCX file and caches successful results. For supported
YouTube file records, it may also attempt to retrieve captions again.

**Parameters:**

- `vaultId` (String): The ID of the vault that owns the bot. Required.
- `botId` (String): The bot's ID. Required.
- `fileId` (String): The **bot file ID**. Required. Use the `id` of an entry in
  `getBotDetails(vaultId, botId).data.files`, not its `userAssetId` (drive file ID).
  The file must belong to the specified bot.

**Example:**

```javascript
try {
  const text = await vault.getBotFileText(
    'your-vault-id',
    'bot-id',
    'bot-file-id'
  );

  console.log('Extracted text:', text);
} catch (error) {
  console.error('Could not retrieve text:', error.code, error.message);
}
```

To find a bot file ID first:

```javascript
try {
  const result = await vault.getBotDetails('your-vault-id', 'bot-id');
  const file = result.data.files[0];

  if (file) {
    const text = await vault.getBotFileText('your-vault-id', 'bot-id', file.id);
    console.log(text);
  }
} catch (error) {
  console.error('Could not retrieve bot file text:', error.code, error.message);
}
```

**Response:**

Resolves to the extracted text as a plain string. There is no `success`,
`message`, or `data` wrapper. The backend serves the content as
`text/plain; charset=utf-8`.

Text availability depends on the file's extracted content and whether the
backend can obtain it. An existing bot file does not guarantee available text.

**Errors:**

- `INVALID_PARAMETER`: A required ID is missing or fails SDK validation.
- `NOT_FOUND`: The bot does not exist or is not owned by the vault user; the file
  does not exist or belongs to another bot; or text cannot be obtained. When
  text is unavailable, the backend message is
  `"Text content not available for this file"`.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.

See **[Error Handling](../Error%20Handling/README.md)** for SDK error types and
other API or network errors.

## Update Bot File Action

### `updateBotFileAction(vaultId, botId, fileId, action)`

Cancels processing or retries a failed bot file.

> The SDK marks this method as an internal helper (`@private`). For application
> code, prefer its public wrappers: `cancelBotFile(vaultId, botId, fileId)` and
> `retryBotFile(vaultId, botId, fileId)`. They delegate to this method with the
> corresponding action.

**Parameters:**

- `vaultId` (String): The ID of the vault that owns the bot. Required.
- `botId` (String): The bot's ID. Required.
- `fileId` (String): The bot file ID from an entry's `id` in
  `getBotDetails(vaultId, botId).data.files`. Required. Use the bot file ID,
  not the drive file's `userAssetId`.
- `action` (String): Required. Must be exactly `'cancel'` or `'retry'`.
  Values are case-sensitive and are not trimmed.

**Actions:**

| Action | Required file state | Behavior |
| --- | --- | --- |
| `'cancel'` | `PROCESSING` | Marks the file `CANCELLED` and clears its processing stage. Background work may continue, but its results are discarded. |
| `'retry'` | `FAILED` | Resets the file to `PROCESSING`, clears its error, increments `retryCount`, and starts processing again in the background. |

Retry is limited to three retry attempts per file. If `failedAt` is recorded,
the request must be within 60 minutes of that failure. The backend also rejects
permanent failures, such as media with no audio, and files without usable
content or a source for retrying. A successful retry response confirms that a
retry started, not that processing finished.

**Cancel example:**

```javascript
try {
  const result = await vault.updateBotFileAction(
    'your-vault-id',
    'bot-id',
    'processing-bot-file-id',
    'cancel'
  );
  console.log(result.message);
} catch (error) {
  console.error('Could not cancel processing:', error.code, error.message);
}
```

**Retry example:**

```javascript
try {
  const result = await vault.updateBotFileAction(
    'your-vault-id',
    'bot-id',
    'failed-bot-file-id',
    'retry'
  );
  console.log(result.message);
  console.log('Status:', result.data.status);
  console.log('Retry count:', result.data.retryCount);
} catch (error) {
  console.error('Could not retry file:', error.code, error.message);
}
```

Equivalent public calls are `vault.cancelBotFile(vaultId, botId, fileId)` and
`vault.retryBotFile(vaultId, botId, fileId)`.

### `cancelBotFile(vaultId, botId, fileId)`

Cancels processing for one bot file. This is the public convenience method for
`updateBotFileAction(..., 'cancel')`.

**Parameters:**

- `vaultId` (String): The vault that owns the bot. Required.
- `botId` (String): The bot ID. Required.
- `fileId` (String): The bot file ID from `getBotDetails(vaultId, botId).data.files`.
  Required. Do not pass the drive asset ID.

The file must currently have `PROCESSING` status. Cancellation marks it
`CANCELLED`, clears its processing stage, releases its reserved transcription
points, and attempts to remove any already-ingested bot data. Background work
may continue, but its results are discarded.

```javascript
try {
  const result = await vault.cancelBotFile(
    'your-vault-id',
    'bot-id',
    'processing-bot-file-id'
  );
  console.log(result.message); // File processing cancelled
} catch (error) {
  console.error('Could not cancel file:', error.code, error.message);
}
```

Returns `{ success: true, message: 'File processing cancelled', data: null }`.
It throws `INVALID_PARAMETER` for invalid IDs, `NOT_FOUND` when the bot or file
is unavailable, and `BAD_REQUEST` when the file is not processing.

### `retryBotFile(vaultId, botId, fileId)`

Retries processing for one failed bot file. This is the public convenience method
for `updateBotFileAction(..., 'retry')`.

**Parameters:**

- `vaultId` (String): The vault that owns the bot. Required.
- `botId` (String): The bot ID. Required.
- `fileId` (String): The bot file ID from `getBotDetails(vaultId, botId).data.files`.
  Required. Do not pass the drive asset ID.

The file must have `FAILED` status, its failure must be within the 60-minute
retry window, and it must have fewer than three previous retries. The backend
also rejects permanent failures and files that cannot be reconstructed from
their source. A successful response means retry processing started; inspect the
file later with `getBotDetails()` for the final status.

```javascript
try {
  const result = await vault.retryBotFile(
    'your-vault-id',
    'bot-id',
    'failed-bot-file-id'
  );
  console.log(result.message); // File retry started
  console.log('New status:', result.data.status);
  console.log('Retry count:', result.data.retryCount);
} catch (error) {
  console.error('Could not retry file:', error.code, error.message);
}
```

Returns `{ success: true, message: 'File retry started', data }`, where `data`
contains the updated serialized bot file. It throws `INVALID_PARAMETER` for
invalid IDs, `NOT_FOUND` when the bot or file is unavailable, and `BAD_REQUEST`
when the file state, retry window, retry count, or source prevents a retry.

**Response:**

Returns the backend response with `success: true` and action-specific fields:

| Action | `message` | `data` |
| --- | --- | --- |
| `'cancel'` | `"File processing cancelled"` | `null` |
| `'retry'` | `"File retry started"` | Updated bot file details, including `id`, `status`, and `retryCount`. |

Use `getBotDetails(vaultId, botId)` to retrieve the file's subsequent status.

**Errors:**

- `INVALID_PARAMETER`: An argument is missing, fails SDK validation, or `action`
  is not exactly `'cancel'` or `'retry'`.
- `BAD_REQUEST`: The file is in the wrong state for the action, retry limits
  have been reached, the failure is permanent, or the file cannot be retried.
- `NOT_FOUND`: The bot is missing or not owned by the vault user, or the file
  is missing or belongs to another bot.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.

Request errors identify the operation as `cancelBotFile` or `retryBotFile`;
argument validation errors identify it as `updateBotFileAction`. See
**[Error Handling](../Error%20Handling/README.md)** for other API or network errors.

## Connect to Bot Chat

See **[Bot Chat Operations](../Bot%20Chat%20Operations/README.md)** for connecting,
joining a bot, sending messages and typing indicators, and disconnecting.

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
