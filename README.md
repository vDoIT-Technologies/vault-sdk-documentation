# Vault SDK Documentation

The **Vault SDK** is a Node.js library for securely managing Vault files, folders, bots, storage plans, wallet data, and bot chat.

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
# Optional controls
VAULT_WS_URL=wss://api.your-service.com/ws
VAULT_ALLOW_INSECURE=false
VAULT_UPLOAD_ROOT=/absolute/path/to/allowed/uploads
VAULT_UPLOAD_HOSTS=additional.storage.example.com
VAULT_TIMEOUT=30000
VAULT_UPLOAD_TIMEOUT=60000
VAULT_UPLOAD_CONCURRENCY=3
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
  VAULT_ALLOW_INSECURE: process.env.VAULT_ALLOW_INSECURE === "true",
  VAULT_UPLOAD_ROOT: process.env.VAULT_UPLOAD_ROOT,
  VAULT_UPLOAD_HOSTS: process.env.VAULT_UPLOAD_HOSTS,
  VAULT_TIMEOUT: Number(process.env.VAULT_TIMEOUT) || 30000,
  VAULT_UPLOAD_TIMEOUT: Number(process.env.VAULT_UPLOAD_TIMEOUT) || undefined,
  VAULT_UPLOAD_CONCURRENCY: Number(process.env.VAULT_UPLOAD_CONCURRENCY) || 3,
});
```

`VAULT_ACCESS_KEY`, `VAULT_SECRET_KEY`, `VAULT_CLIENT_API_KEY`, and
`VAULT_BASE_URL` are required. `VAULT_WS_URL` is required only for the general
Vault WebSocket. The constructor throws `MISSING_CONFIG` when a required value
is absent.

These values are constructor options; the SDK does not load a `.env` file by
itself. `VAULT_ALLOW_INSECURE` must be passed as the boolean `true` to take
effect.

The SDK requires HTTPS/WSS for non-local endpoints. Set
`VAULT_ALLOW_INSECURE: true` only for a local test server. Upload paths can be
restricted with `VAULT_UPLOAD_ROOT`; presigned upload hosts are checked against
the built-in allowlist and `VAULT_UPLOAD_HOSTS`.

Every API request is signed by the SDK with the configured access and secret
keys. The client API key is sent automatically as an authentication header; do
not put any of these credentials in user-controlled method arguments. Secret
configuration values are non-enumerable and are redacted by `console.log()` and
`JSON.stringify()`.

Run the SDK in a trusted server-side process. Do not ship `VAULT_SECRET_KEY`,
`VAULT_ACCESS_KEY`, or `VAULT_CLIENT_API_KEY` to a browser or mobile client.

## First integration

Use this sequence when adding Vault to an application:

1. Obtain the three Vault credentials and API base URL from the Vault administrator.
2. Initialize one `Vault` instance in your server application.
3. Create a Vault user or import an existing Vault to obtain a `vaultId`.
4. Pass that `vaultId` to the file, folder, bot, storage, wallet, or chat methods.
5. Catch `ValidationError` and `VaultError` around SDK calls.

```javascript
import Vault, { VaultError, ValidationError } from "vault-sdk-dev";

const vault = new Vault({
  VAULT_ACCESS_KEY: process.env.VAULT_ACCESS_KEY,
  VAULT_SECRET_KEY: process.env.VAULT_SECRET_KEY,
  VAULT_CLIENT_API_KEY: process.env.VAULT_CLIENT_API_KEY,
  VAULT_BASE_URL: process.env.VAULT_BASE_URL,
});

try {
  // Pass a platform ID when your client is not already linked to one.
  const user = await vault.createVault("user@example.com");
  const vaultId = user.data.vaultId;

  const files = await vault.getAllFiles(vaultId);
  console.log(files.data);
} catch (error) {
  if (error instanceof ValidationError || error instanceof VaultError) {
    console.error(error.code, error.message);
  }
  throw error;
}
```

Most HTTP methods return the server's `{ success, message, data }` response.
The main exceptions are WebSocket methods, which use events and connection
metadata, and `getBotFileText()`, which returns extracted text directly.

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

### Updating a Bot

Apply a partial update to a bot's editable settings:

```javascript
const result = await vault.updateBot('your-vault-id', 'bot-id', {
  description: 'Answers customer questions clearly',
  wordLimit: 300,
});
console.log(result.data);
```

See **[Bot Operations](./Bot%20Operations/README.md#update-bot)** for supported
fields, limits, and errors.

### Quoting Transcription Cost

Estimate Twin Points before uploading or linking media to a bot:

```javascript
const quote = await vault.quoteTranscription('your-vault-id', 'bot-id', {
  files: [{ name: 'call.mp3', size: 12_000_000, durationSeconds: 905 }],
});
console.log(quote.data.estimatedPoints, quote.data.enough);
```

See **[Bot Operations](./Bot%20Operations/README.md#quote-transcription)** for
payload fields, folder scanning, response data, and errors.

### Deleting a Bot

Delete a bot by passing its ID first and the owning vault ID second:

```javascript
await vault.deleteBot('bot-id', 'your-vault-id');
```

The bot and its knowledge are removed, while linked drive files and folders are
preserved. See **[Bot Operations](./Bot%20Operations/README.md#delete-bot)** for
the response, warnings, and error handling.

### Removing a Bot Asset

Remove a file or unlink a folder from a bot:

```javascript
await vault.removeBotAsset('your-vault-id', 'bot-id', 'file', 'bot-file-id');
await vault.removeBotAsset('your-vault-id', 'bot-id', 'folder', 'folder-id');
```

Pass `{ permanent: true }` for a file to delete its drive copy, or
`{ permanent: true, keepTranscript: true }` to retain its extracted transcript.
See **[Bot Operations](./Bot%20Operations/README.md#remove-a-bot-asset)** for
the complete behavior and response fields.

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

### Exporting Bot Sessions

Export one or more saved sessions to the drive or into a bot's knowledge set:

```javascript
const result = await vault.exportBotSessions(
  'your-vault-id',
  'bot-id',
  ['session-id-1', 'session-id-2'],
  'drive'
);
console.log(result.data.fileName);
```

Use `'brain'` as the save option to ingest the export into the source bot, or
pass a fifth `targetBotId` to ingest it into another bot owned by the same user.
See **[Bot Session Operations](./Bot%20Session%20Operations/README.md#export-bot-sessions)**
for response fields and errors.

### Deleting Bot Sessions

Permanently delete one or more sessions and their messages:

```javascript
const result = await vault.deleteBotSessions(
  'your-vault-id',
  'bot-id',
  ['session-id-1', 'session-id-2']
);
console.log(result.data.deletedCount);
```

The method also accepts a single session ID. See
**[Bot Session Operations](./Bot%20Session%20Operations/README.md#delete-bot-sessions)**
for filtering behavior, responses, and errors.

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

### Getting Wallet Information

Retrieve the authenticated vault user's Twin Points balance and wallet status:

```javascript
const wallet = await vault.getWalletInfo('your-vault-id');
console.log(wallet.data.points, wallet.data.status);
```

See **[Wallet Operations](./Wallet%20Operations/README.md#get-wallet-info)** for
response fields and errors.

### Getting Transaction History

Retrieve paginated wallet transactions, optionally filtered by category:

```javascript
const history = await vault.getTransactionHistory('your-vault-id', {
  page: 1,
  limit: 20,
  category: 'credit',
});
console.log(history.data.transactions, history.data.pagination);
```

See **[Wallet Operations](./Wallet%20Operations/README.md#get-transaction-history)**
for query options, pagination fields, and errors.

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
- **[WebSocket Operations](./WebSocket%20Operations/README.md)**: Connect to the general Vault real-time event stream.
- **[Bot Session Operations](./Bot%20Session%20Operations/README.md)**: Retrieve and export saved bot chat sessions and message history.
- **[Storage Operations](./Storage%20Operations/README.md)**: Storage usage, plans, subscriptions, and upcoming plans.
- **[Wallet Operations](./Wallet%20Operations/README.md)**: Retrieve Twin Points wallet balance and status.
- **[Authentication Operations](./Authentication%20Operations/README.md)**: Create and redeem launch tokens for Vault and bot-chat authentication.
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
| `renameFile(vaultId, itemId, newName)` (alias) | [File Operations](./File%20Operations/README.md#renameitemvaultid-itemid-newname) |
| `addToStarred(vaultId, fileId, isStarred)` | [File Operations](./File%20Operations/README.md) |
| `getStarredFiles(vaultId)` | [File Operations](./File%20Operations/README.md) |
| `createFolder(vaultId, folderName, parentId?)` | [Folder Operations](./Folder%20Operations/README.md) |
| `deleteFolder(vaultId, folderId)` | [Folder Operations](./Folder%20Operations/README.md) |
| `createBot(vaultId, bot)` | [Bot Operations](./Bot%20Operations/README.md) |
| `deleteBot(botId, vaultId)` | [Bot Operations](./Bot%20Operations/README.md#delete-bot) |
| `removeBotAsset(vaultId, botId, assetType, assetId, options?)` | [Bot Operations](./Bot%20Operations/README.md#remove-a-bot-asset) |
| `getBotDetails(vaultId, botId?)` | [Bot Operations](./Bot%20Operations/README.md#get-bot-details) |
| `updateBot(vaultId, botId, updates)` | [Bot Operations](./Bot%20Operations/README.md#update-bot) |
| `quoteTranscription(vaultId, botId, payload?)` | [Bot Operations](./Bot%20Operations/README.md#quote-transcription) |
| `getBotSessions(vaultId, botId, sessionId?)` | [Bot Session Operations](./Bot%20Session%20Operations/README.md#get-bot-sessions) |
| `deleteBotSessions(vaultId, botId, sessionIds)` | [Bot Session Operations](./Bot%20Session%20Operations/README.md#delete-bot-sessions) |
| `exportBotSessions(vaultId, botId, sessionIds, saveOption, targetBotId?)` | [Bot Session Operations](./Bot%20Session%20Operations/README.md#export-bot-sessions) |
| `getBotFileText(vaultId, botId, fileId)` | [Bot Operations](./Bot%20Operations/README.md#get-bot-file-text) |
| `updateBotFileAction(vaultId, botId, fileId, action)` (internal helper) | [Bot Operations](./Bot%20Operations/README.md#update-bot-file-action) |
| `cancelBotFile(vaultId, botId, fileId)` | [Bot Operations](./Bot%20Operations/README.md#cancelbotfilevaultid-botid-fileid) |
| `retryBotFile(vaultId, botId, fileId)` | [Bot Operations](./Bot%20Operations/README.md#retrybotfilevaultid-botid-fileid) |
| `connectToBotChat(vaultId, options?)` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#connect-to-bot-chat) |
| `joinBotChat(botId, sessionId?)` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#join-bot-chat) |
| `sendBotChatMessage(message, history?)` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#send-a-chat-message) |
| `sendBotChatTyping()` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#send-a-typing-indicator) |
| `disconnectBotChat()` | [Bot Chat Operations](./Bot%20Chat%20Operations/README.md#disconnect-from-bot-chat) |
| `connectToWebsocket()` | [WebSocket Operations](./WebSocket%20Operations/README.md#connect-to-vault-events) |
| `uploadFilesToBot(files, vaultId, botId)` | [Bot Operations](./Bot%20Operations/README.md#upload-files-to-a-bot) |
| `addDriveFilesToBot(vaultId, botId, fileIds)` | [Bot Operations](./Bot%20Operations/README.md#add-drive-files-to-a-bot) |
| `addDriveFoldersToBot(vaultId, botId, folderIds)` | [Bot Operations](./Bot%20Operations/README.md#add-drive-folders-to-a-bot) |
| `getStorageDetails(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `getWalletInfo(vaultId)` | [Wallet Operations](./Wallet%20Operations/README.md#get-wallet-info) |
| `getTransactionHistory(vaultId, query?)` | [Wallet Operations](./Wallet%20Operations/README.md#get-transaction-history) |
| `getAllPlans(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `buyPlan(vaultId, priceId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `getSubscriptions(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `cancelSubscription(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `createUpcomingPlan(vaultId, priceId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `cancelUpcomingPlan(vaultId)` | [Storage Operations](./Storage%20Operations/README.md) |
| `createVault(email, platformId?)` | [User Operations](./User%20Operations/README.md) |
| `importVault(vaultId, platformId?)` | [User Operations](./User%20Operations/README.md) |
| `createVaultLaunchToken(vaultId, options?)` | [Authentication Operations](./Authentication%20Operations/README.md#create-a-vault-launch-token) |
| `redeemVaultLaunchToken(launchToken)` | [Authentication Operations](./Authentication%20Operations/README.md#redeem-a-launch-token) |
| `createBotChatAccessToken(vaultId, options?)` | [Authentication Operations](./Authentication%20Operations/README.md#create-a-bot-chat-access-token) |

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
