# Authentication Operations

This section documents SDK authentication helpers. It does not describe Vault
web-app navigation or other platform-user implementations.

## Create a Vault launch token

### `createVaultLaunchToken(vaultId, options?)`

Creates a short-lived launch token and URL for handing a user from an SDK
integration into Vault.

**Parameters:**

- `vaultId` (String): Vault ID to authenticate. Required.
- `options` (Object, optional): Launch context.
  - `returnTo` (String, optional): Internal path or allowed `http(s)` URL.
  - `clientId`, `adminId`, `sourceUserId` (String, optional): Context values
    forwarded to the Vault launch session.

```javascript
const launch = await vault.createVaultLaunchToken('your-vault-id', {
  returnTo: '/drive',
});

// Standard response fields:
const launchUrl = launch.data.launchUrl;
const launchToken = launch.data.launchToken;
```

`returnTo` is validated by the SDK to reject unsafe paths, backslashes, control
characters, and unsupported URL schemes. Treat the token and URL as credentials:
do not log them or store them in persistent client-side state.

## Redeem a launch token

### `redeemVaultLaunchToken(launchToken)`

Redeems a one-time launch token for a normal Vault access token.

```javascript
const redeemed = await vault.redeemVaultLaunchToken(launchToken);
const accessToken = redeemed.data.user.accessToken;
```

The token is short-lived and single-use. A failed or already-used token must not
be retried indefinitely.

## Create a bot-chat access token

### `createBotChatAccessToken(vaultId, options?)`

Creates and redeems a launch token, returning only the Vault access token needed
by the bot-chat WebSocket.

```javascript
const accessToken = await vault.createBotChatAccessToken('your-vault-id');
await vault.connectToBotChat('your-vault-id', {
  token: accessToken,
  botId: 'bot-id',
});
```

`options` has the same launch context fields as
`createVaultLaunchToken()`. `connectToBotChat()` performs this flow
automatically when neither `token` nor `launchToken` is supplied.

## Create or import Vault users

`createVault(email, platformId?)` and `importVault(vaultId, platformId?)` are
SDK methods documented in **[User Operations](../User%20Operations/README.md)**.
The optional `platformId` is an SDK request parameter. When omitted, the server resolves the platform associated with the
configured client API key.
