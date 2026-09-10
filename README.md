# Vault SDK Documentation

The **Vault SDK** is a Node.js library designed to provide seamless integration with the Vault Service. It allows developers to manage files, folders, and storage plans, as well as interact with the Twin Protocol backend for secure vault operations.

## Installation

To use the Vault SDK, you must first install it in your Node.js environment.

```bash
# Development
$ npm install vault-sdk-dev
```

*(Note: Adjust the package name if using staging or production versions, e.g., `vault-sdk-staging` or `vault-sdk-prod`, if applicable.)*

## Configuration

Before initializing the SDK, ensure you have the following environment variables configured. These credentials can be obtained from your **Client Admin Portal**.

```bash
VAULT_ACCESS_KEY=your-access-key
VAULT_SECRET_KEY=your-secret-key
VAULT_CLIENT_API_KEY=your-client-api-key
VAULT_BASE_URL=https://api.your-service.com
```

## Usage

### Initialization

Import and initialize the SDK in your application:

```javascript
import Vault from "vault-sdk-dev";

const vault = new Vault({
  VAULT_ACCESS_KEY: process.env.VAULT_ACCESS_KEY,
  VAULT_SECRET_KEY: process.env.VAULT_SECRET_KEY,
  VAULT_CLIENT_API_KEY: process.env.VAULT_CLIENT_API_KEY,
  VAULT_BASE_URL: process.env.VAULT_BASE_URL,
  VAULT_WS_URL: process.env.VAULT_WS_URL,
});
```

All parameters are required — the constructor throws a `VaultError` with code
`MISSING_CONFIG` listing any that are absent.

### Uploading a File

Uploading takes a single call — pass a path, a `{ buffer, name }` object, or a
`File`/`Blob`, and the SDK handles hashing, the presigned URL, the transfer, and
registration for you:

```javascript
const result = await vault.uploadFile('./report.pdf', 'your-vault-id');
```

See **[File Operations](./File%20Operations/README.md)** for all accepted file forms
and multi-file uploads.

### Getting Bot Details

Retrieve a bot with its associated files and linked folders, or omit `botId`
to retrieve all bots in the vault:

```javascript
const result = await vault.getBotDetails('your-vault-id', 'bot-id');
console.log(result.data);

const allBots = await vault.getBotDetails('your-vault-id');
console.log(allBots.data);
```

`vaultId` is required as the first argument. See
**[Bot Operations](./Bot%20Operations/README.md#get-bot-details)** for parameters,
response fields, and error-handling examples.

### Deleting a Bot

Delete a bot by passing its ID first and the owning vault ID second:

```javascript
await vault.deleteBot('bot-id', 'your-vault-id');
```

The bot and its knowledge are removed, while linked drive files and folders are
preserved. See **[Bot Operations](./Bot%20Operations/README.md#delete-bot)** for
the response, warnings, and error handling.

### Getting Bot File Text

Retrieve extracted text using a bot file's `id` from `getBotDetails().data.files`:

```javascript
const text = await vault.getBotFileText(
  'your-vault-id',
  'bot-id',
  'bot-file-id'
);
console.log(text);
```

This method returns a plain string. See
**[Bot Operations](./Bot%20Operations/README.md#get-bot-file-text)** for file ID
selection, text availability, and error handling.

### Cancelling or Retrying Bot File Processing

Use `cancelBotFile(vaultId, botId, fileId)` for a processing file or
`retryBotFile(vaultId, botId, fileId)` for an eligible failed file. Both public
methods delegate to the internal helper `updateBotFileAction(vaultId, botId,
fileId, action)`, which accepts `'cancel'` or `'retry'`.

```javascript
await vault.cancelBotFile('your-vault-id', 'bot-id', 'processing-file-id');
const retry = await vault.retryBotFile('your-vault-id', 'bot-id', 'failed-file-id');
console.log(retry.data.status, retry.data.retryCount);
```

See **[Bot Operations](./Bot%20Operations/README.md#update-bot-file-action)** for
examples, retry limits, responses, and error handling.

### Connecting to Bot Chat

Open a live chat connection and automatically join a bot:

```javascript
vault.on('bot_chat_chat_history', (payload) => {
  console.log('Joined bot:', payload.botName);
});
vault.on('bot_chat_error', (payload) => console.error(payload.message));
vault.on('bot_chat_stream_error', (error) => console.error(error.message));

await vault.connectToBotChat('your-vault-id', { botId: 'bot-id' });
```

The SDK obtains an access token automatically unless one is supplied. The
promise resolves when the socket opens; chat events report joining the bot and
receiving replies. Close the connection with `vault.disconnectBotChat()` when
finished. See **[Bot Chat Operations](./Bot%20Chat%20Operations/README.md#connect-to-bot-chat)**
for options, a messaging example, events, and error handling.

### Uploading Files to a Bot

Upload a single file or an array to a bot's dedicated folder and start knowledge
processing in the background:

```javascript
const result = await vault.uploadFilesToBot(
  ['./faq.pdf', './product-guide.txt'],
  'your-vault-id',
  'bot-id'
);
```

Inspect `result.data.results` for individual upload outcomes. Partial failures
are returned in the response; if all files fail, the method throws
`UPLOAD_FAILED` with details in `error.data.results`. See
**[Bot Operations](./Bot%20Operations/README.md#upload-files-to-a-bot)** for
parameters, response fields, and error-handling examples.

### Adding Existing Drive Files to a Bot

Link files already stored in the vault to a bot's knowledge set:

```javascript
const result = await vault.addDriveFilesToBot(
  'your-vault-id',
  'bot-id',
  ['file-id-1', 'file-id-2']
);
```

`fileIds` also accepts a single file ID. Check `result.data.results` for each
entry's `success` boolean; already-linked files count as successes. If every
file fails, the SDK throws `BAD_REQUEST` with per-file details in
`error.data.data.results`. See
**[Bot Operations](./Bot%20Operations/README.md#add-drive-files-to-a-bot)** for
examples, response fields, and limits.

### Adding Existing Drive Folders to a Bot

Attach drive folders and ingest compatible files, including files in nested
folders:

```javascript
const result = await vault.addDriveFoldersToBot(
  'your-vault-id',
  'bot-id',
  ['folder-id-1', 'folder-id-2']
);
```

`folderIds` also accepts a single folder ID. Inspect `result.data.results` for
folder outcomes and each successful entry's `linkedFiles` and `brainIngestion`:
a folder can attach successfully even when file ingestion fails. If every folder
entry fails, the SDK throws `BAD_REQUEST` with details in
`error.data.data.results`. See
**[Bot Operations](./Bot%20Operations/README.md#add-drive-folders-to-a-bot)** for
examples, response fields, and limits.

### Handling Errors

Always use `try...catch` blocks to handle SDK errors. The SDK provides `VaultError` and `ValidationError` for precise error catching.

```javascript
import { VaultError, ValidationError } from "vault-sdk-dev";

try {
  await vault.getAllFiles('your-vault-id');
} catch (error) {
  if (error instanceof ValidationError) {
    console.error("Invalid Parameter:", error.param);
  } else if (error instanceof VaultError) {
    console.error("API Error Code:", error.code);
  }
}
```

See the **[Error Handling](./Error%20Handling/README.md)** section for more details.

## Structure

The documentation is organized into the following sections:

- **[File Operations](./File%20Operations/README.md)**: Upload, search, rename, star, and delete files.
- **[Folder Operations](./Folder%20Operations/README.md)**: Create, rename, and delete folders.
- **[Bot Operations](./Bot%20Operations/README.md)**: Create bots, retrieve details, upload files, and add existing drive files and folders to bot knowledge.
- **[Bot Chat Operations](./Bot%20Chat%20Operations/README.md)**: Connect to live chat, join bots, send messages and typing indicators, and disconnect.
- **[Storage Operations](./Storage%20Operations/README.md)**: Storage usage, plans, subscriptions, and upcoming plans.
- **[User Operations](./User%20Operations/README.md)**: Create vaults for users and import existing vaults.
- **[Error Handling](./Error%20Handling/README.md)**: Detailed guide on handling SDK errors and codes.

## API Reference

| Method | Section |
| --- | --- |
| `uploadFile(file, vaultId, parentId?)` | [File Operations](./File%20Operations/README.md) |
| `uploadFiles(files, vaultId, parentId?)` | [File Operations](./File%20Operations/README.md) |
| `getFiles(vaultId, query?)` | [File Operations](./File%20Operations/README.md) |
| `getAllFiles(vaultId)` | [File Operations](./File%20Operations/README.md) |
| `deleteFile(vaultId, fileId)` | [File Operations](./File%20Operations/README.md) |
| `renameItem(vaultId, itemId, newName)` | [File Operations](./File%20Operations/README.md) |
| `addToStarred(vaultId, fileId, isStarred)` | [File Operations](./File%20Operations/README.md) |
| `getStarredFiles(vaultId)` | [File Operations](./File%20Operations/README.md) |
| `createFolder(vaultId, folderName, parentId?)` | [Folder Operations](./Folder%20Operations/README.md) |
| `deleteFolder(vaultId, folderId)` | [Folder Operations](./Folder%20Operations/README.md) |
| `createBot(vaultId, bot)` | [Bot Operations](./Bot%20Operations/README.md) |
| `deleteBot(botId, vaultId)` | [Bot Operations](./Bot%20Operations/README.md#delete-bot) |
| `getBotDetails(vaultId, botId?)` | [Bot Operations](./Bot%20Operations/README.md#get-bot-details) |
| `getBotFileText(vaultId, botId, fileId)` | [Bot Operations](./Bot%20Operations/README.md#get-bot-file-text) |
| `updateBotFileAction(vaultId, botId, fileId, action)` (internal helper) | [Bot Operations](./Bot%20Operations/README.md#update-bot-file-action) |
| `cancelBotFile(vaultId, botId, fileId)` | [Bot Operations](./Bot%20Operations/README.md#cancelbotfilevaultid-botid-fileid) |
| `retryBotFile(vaultId, botId, fileId)` | [Bot Operations](./Bot%20Operations/README.md#retrybotfilevaultid-botid-fileid) |
| `connectToBotChat(vaultId, options?)` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#connect-to-bot-chat) |
| `joinBotChat(botId, sessionId?)` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#join-bot-chat) |
| `sendBotChatMessage(message, history?)` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#send-a-chat-message) |
| `sendBotChatTyping()` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#send-a-typing-indicator) |
| `disconnectBotChat()` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#disconnect-from-bot-chat) |
| `uploadFilesToBot(files, vaultId, botId)` | [Bot Operations](./Bot%20Operations/README.md#upload-files-to-a-bot) |
| `addDriveFilesToBot(vaultId, botId, fileIds)` | [Bot Operations](./Bot%20Operations/README.md#add-drive-files-to-a-bot) |
| `addDriveFoldersToBot(vaultId, botId, folderIds)` | [Bot Operations](./Bot%20Operations/README.md#add-drive-folders-to-a-bot) |
| `getStorageDetails(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `getAllPlans(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `buyPlan(vaultId, priceId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `getSubscriptions(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `cancelSubscription(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `createUpcomingPlan(vaultId, priceId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `cancelUpcomingPlan(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `createVault(email, platformId?)` | [User Operations](./User%20Operations/README.md) |
| `importVault(vaultId, platformId?)` | [User Operations](./User%20Operations/README.md) |

## Migrating from earlier versions

If you are upgrading an integration written against an older release, note these
breaking changes:

| Previously | Now |
| --- | --- |
| `createPlatformUser(email, platformId)` | `createVault(email, platformId?)` — `platformId` is now optional |
| `getMedia(vaultId)` | Removed. Use `getAllFiles(vaultId)` or `getFiles(vaultId, query)` |
| `getPresignedUrl({...})` + manual `PUT` + `registerUpload({...})` | Removed as public methods. `uploadFile()` performs all three steps internally |
| `renameFile(vaultId, itemId, newName)` | `renameItem(vaultId, itemId, newName)`. `renameFile()` still works as an alias |
| `uploadFiles()` required `{ name, buffer }` objects | Entries may also be a path string, `{ path }`, or a `File`/`Blob`; `type` is optional |
| `importVault(vaultId, platformId)` required `platformId` | `platformId` is optional |
