# User Operations

This section provides details on creating vaults for users and importing existing vaults.

## Create a Vault for a User

### `createVault(email, platformId)`

Creates a vault for a user identified by their email address. This is typically the
starting point for integrating a user into the Vault system.

The call is **idempotent**: if a user already exists for that email, they are linked
to your client API key and returned instead of raising a conflict, so it is safe to
call on every sign-in.

> **Renamed:** this method was previously called `createPlatformUser()`. That name is
> no longer available — use `createVault()`.

**Parameters:**

- `email` (String): The email address of the user.
- `platformId` (String, optional): The platform to associate the user with. Omit it
  to create a platform-less SDK user linked directly to your client.

**Example:**

```javascript
try {
  // Associated with a platform
  const newUser = await vault.createVault('user@example.com', 'your-platform-id');
  console.log('User created:', newUser);

  // Platform-less SDK user
  const sdkUser = await vault.createVault('user@example.com');
  console.log('SDK user created:', sdkUser);
} catch (error) {
  console.error('Error creating vault:', error);
}
```

**Response:**

Returns the created (or existing) user, including the `vaultId` you pass to the
file, folder, and storage methods.

## Import an Existing Vault

### `importVault(vaultId, platformId)`

Imports an existing vault into the current platform context. This is useful for
linking a user's pre-existing data to a new application or service instance.

**Parameters:**

- `vaultId` (String): The ID of the vault to import.
- `platformId` (String, optional): The target platform. When omitted, SDK access is
  enabled and your client is linked directly to the user, without a platform.

**Example:**

```javascript
try {
  // Import into a platform
  const importResult = await vault.importVault('existing-vault-id', 'your-platform-id');
  console.log('Vault imported successfully:', importResult);

  // Import without a platform
  const sdkImport = await vault.importVault('existing-vault-id');
  console.log('Vault linked to client:', sdkImport);
} catch (error) {
  console.error('Import failed:', error);
}
```

**Response:**

Returns a confirmation object or updated user details reflecting the imported vault association.
