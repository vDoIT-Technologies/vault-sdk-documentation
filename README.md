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
VAULT_WS_URL=wss://api.your-service.com/ws
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

### WebSocket Connection

To listen for real-time updates:

```javascript
await vault.connectToWebsocket();

vault.on('message', (data) => {
  console.log('Received message:', data);
});

vault.on('stream_error', (error) => {
  console.error('WebSocket error:', error);
});
```

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

- **[File Operations](./File%20Operations/README.md)**: Manage files (upload, download, delete, rename, etc.).
- **[Folder Operations](./Folder%20Operations/README.md)**: Manage folders (create, delete).
- **[Storage Operations](./Storage%20Operations/README.md)**: Manage storage plans and subscriptions.
- **[User Operations](./User%20Operations/README.md)**: Manage vault users and imports.
- **[Error Handling](./Error%20Handling/README.md)**: Detailed guide on handling SDK errors and codes.
