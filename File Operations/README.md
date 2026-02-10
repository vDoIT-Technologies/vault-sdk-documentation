# File Operations

This section covers all operations related to file management within the Vault.

## Upload Files

You have two options for uploading files:
1. **Automatic Upload**: Use `uploadFiles` for a simple, all-in-one process.
2. **Manual Upload Flow**: Use explicit steps to generate a link, upload, and register. This is useful for large files or custom upload handling.

### Option 1: Automatic Upload

#### `uploadFiles(files, vaultId, parentId)`

Uploads multiple files automatically.

**Parameters:**

- `files` (Array): An array of file objects. Each file object should have:
  - `name` (String): The name of the file.
  - `buffer` (Buffer): The file content.
  - `type` (String, optional): The MIME type of the file.
- `vaultId` (String): The ID of the vault where files will be uploaded.
- `parentId` (String, optional): The ID of the parent folder. Defaults to `null` (root).

**Example:**

```javascript
const files = [
  { 
    name: 'document.pdf', 
    buffer: fs.readFileSync('path/to/document.pdf'), 
    type: 'application/pdf' 
  }
];

const results = await vault.uploadFiles(files, 'your-vault-id');
console.log(results);
```

### Option 2: Manual Upload Flow

For greater control, you can break the upload process into three steps:

#### Step 1: Get Presigned URL

Generate a secure URL to upload your file.

**`getPresignedUrl({ vaultId, fileName, fileType, fileSize, contentHash, folderId })`**

```javascript
const crypto = require('crypto');

const fileBuffer = fs.readFileSync('large-video.mp4');
const fileSize = fileBuffer.length;
const contentHash = crypto.createHash('sha256').update(fileBuffer).digest('hex');

const presignedData = await vault.getPresignedUrl({
  vaultId: 'your-vault-id',
  fileName: 'large-video.mp4',
  fileType: 'video/mp4',
  fileSize: fileSize,
  contentHash: contentHash,
  folderId: 'optional-folder-id' // or null
});

const { url, key, contentType, sanitizedName } = presignedData.data;
```

#### Step 2: Upload to Storage

Use the generated URL to upload the file content directly. You can use any HTTP client (like `axios`, `fetch`, etc.).

```javascript
const axios = require('axios');

await axios.put(url, fileBuffer, {
  headers: {
    "Content-Type": contentType,
    "x-amz-meta-original-filename": sanitizedName,
    "x-amz-meta-content-hash": contentHash,
    "x-amz-meta-user-id": 'your-vault-id',
    "x-amz-meta-folder-id": 'optional-folder-id' || "root",
    "x-amz-meta-file-size": fileSize.toString(),
  }
});
```

#### Step 3: Register Upload

Confirm the upload with the Vault backend.

**`registerUpload({ vaultId, fileName, filebaseKey, fileSize, contentHash, folderId })`**

```javascript
const registerForMyFile = await vault.registerUpload({
  vaultId: 'your-vault-id',
  fileName: 'large-video.mp4',
  filebaseKey: key, // Received from getPresignedUrl
  fileSize: fileSize,
  contentHash: contentHash,
  folderId: 'optional-folder-id' // or null
});

console.log('Upload Registered:', registerForMyFile);
```

---

## Retrieve Files

### `getFiles(vaultId, query)`

Searches for files or lists files based on a query.

**Parameters:**

- `vaultId` (String): The ID of the vault.
- `query` (String, optional): Search query string.

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

### `getMedia(vaultId)`

Fetches all media files associated with the vault ID.

**Parameters:**

- `vaultId` (String): The ID of the vault.

**Example:**

```javascript
const mediaFiles = await vault.getMedia('your-vault-id');
console.log(mediaFiles);
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

### `renameFile(vaultId, itemId, newName)`

Renames a file or a folder.

**Parameters:**

- `vaultId` (String): The ID of the vault.
- `itemId` (String): The unique ID of the file or folder.
- `newName` (String): The new name for the item.

**Example:**

```javascript
const response = await vault.renameFile('your-vault-id', 'item-id', 'New Name.txt');
console.log(response);
```

## Starred Files

### `addToStarred(vaultId, fileId, isStarred)`

Marks or unmarks a file as starred (favorite).

**Parameters:**

- `vaultId` (String): The ID of the vault.
- `fileId` (String): The unique ID of the file.
- `isStarred` (Boolean): `true` to star, `false` to unstar.

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
