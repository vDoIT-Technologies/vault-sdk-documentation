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
- **[Bot Operations](./Bot%20Operations/README.md)**: Create bots and their dedicated folders.
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
