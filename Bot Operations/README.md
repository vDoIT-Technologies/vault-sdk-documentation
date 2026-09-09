# Bot Operations

This section provides details on creating bots within the Vault.

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
