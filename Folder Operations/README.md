# Folder Operations

This section provides details on creating and managing folders within the Vault.

## Create Folder

### `createFolder(vaultId, folderName, parentId)`

Creates a new folder within the specified vault. You can optionally specify a parent directory to create a subfolder.

**Parameters:**

- `vaultId` (String): The ID of the vault where the folder will be created.
- `folderName` (String): The name of the new folder.
- `parentId` (String, optional): The ID of the parent folder. Defaults to `null` (root).

**Example:**

```javascript
// Create a folder in the root directory
const rootFolder = await vault.createFolder('your-vault-id', 'Documents');
console.log(rootFolder);

// Create a subfolder, using the ID returned by the call above
const subFolder = await vault.createFolder(
  'your-vault-id',
  'Project Alpha',
  'parent-folder-id'
);
```

**Response:**

Returns an object containing the details of the newly created folder, including its unique ID.

## Rename Folder

### `renameItem(vaultId, itemId, newName)`

`renameItem()` works on folders as well as files — pass the folder's ID as `itemId`.

**Example:**

```javascript
await vault.renameItem('your-vault-id', 'folder-id', 'Archived Documents');
```

See **[File Operations](../File%20Operations/README.md#renameitemvaultid-itemid-newname)** for full details.

## Delete Folder

### `deleteFolder(vaultId, folderId)`

Permanently deletes a folder and its contents.

**Parameters:**

- `vaultId` (String): The ID of the vault from which to delete the folder.
- `folderId` (String): The unique ID of the folder to delete.

**Example:**

```javascript
const response = await vault.deleteFolder('your-vault-id', 'folder-id-to-delete');
console.log(response); // Confirmation message or status
```

**Note:** Deleting a folder will also delete all files and subfolders contained within it.
