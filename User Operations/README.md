# User Operations

This section provides details on managing users and importing existing vaults.

## Create a New User

### `createPlatformUser(email, platformId)`

Creates a new platform user associated with a specific email address. This is typically the starting point for integrating a user into the Vault system.

**Parameters:**

- `email` (String): The email address of the user.
- `platformId` (String): The unique identifier for the platform (e.g., your app's ID).

**Example:**

```javascript
try {
  const newUser = await vault.createPlatformUser('user@example.com', 'your-platform-id');
  console.log('User created:', newUser);
} catch (error) {
  console.error('Error creating user:', error);
}
```

**Response:**

Returns the newly created user object, which may include a `vaultId` or other identifiers.

## Import an Existing Vault

### `importVault(vaultId, platformId)`

Imports an existing vault into the current platform context. This is useful for linking a user's pre-existing data to a new application or service instance.

**Parameters:**

- `vaultId` (String): The ID of the vault to import.
- `platformId` (String): The unique identifier for the platform importing the vault.

**Example:**

```javascript
try {
  const importResult = await vault.importVault('existing-vault-id', 'your-platform-id');
  console.log('Vault imported successfully:', importResult);
} catch (error) {
  console.error('Import failed:', error);
}
```

**Response:**

Returns a confirmation object or updated user details reflecting the imported vault association.
