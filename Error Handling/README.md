# Error Handling

The Vault SDK provides a robust error-handling mechanism to help you identify and resolve issues quickly. All errors thrown by the SDK are categorized into two main types: `ValidationError` and `VaultError`.

## Error Types

### `ValidationError`

Thrown when the parameters passed to an SDK method fail validation (e.g., a required parameter is missing or has the wrong type).

**Properties:**
- `message`: A descriptive message indicating what failed.
- `code`: Always `"INVALID_PARAMETER"`.
- `param`: The name of the parameter that failed validation.
- `operation`: The name of the method where the error occurred.

### `VaultError`

Thrown for API errors, network issues, or internal SDK failures.

**Properties:**
- `message`: Descriptive error message.
- `code`: A unique string identifier for the error (see [Error Codes](#error-codes)).
- `status`: The HTTP status code (if applicable).
- `operation`: The name of the method or endpoint where the error occurred.
- `data`: The raw error response data from the server (if available).

---

## Best Practices

Always wrap SDK calls in `try...catch` blocks to handle potential errors gracefully.

```javascript
import Vault, { VaultError, ValidationError } from "vault-sdk-dev";

const vault = new Vault({ ... });

try {
  await vault.uploadFile(file, "vault-id");
} catch (error) {
  if (error instanceof ValidationError) {
    // Parameter validation failed
    console.error(`Validation Failed in ${error.operation}: ${error.message}`);
    console.error(`Missing/Invalid Parameter: ${error.param}`);
  } else if (error instanceof VaultError) {
    // API or Network error
    console.error(`Vault Error (${error.code}): ${error.message}`);
    if (error.status) console.error(`HTTP Status: ${error.status}`);
  } else {
    // Unexpected error
    console.error("An unexpected error occurred:", error);
  }
}
```

---

## Error Codes

The following table lists common error codes returned in `VaultError.code`:

| Code | Description |
|------|-------------|
| `MISSING_CONFIG` | Required configuration parameter (e.g., `VAULT_ACCESS_KEY`) not provided in the constructor. |
| `INVALID_PARAMETER` | Method parameter failed validation. |
| `BAD_REQUEST` | The server rejected the request (HTTP 400). Check your payload. |
| `UNAUTHORIZED` | Authentication failed (HTTP 401). Check your Access Key and Secret Key. |
| `FORBIDDEN` | Your API key does not have permission for this operation (HTTP 403). |
| `NOT_FOUND` | The requested resource (vault, file, folder) does not exist (HTTP 404). |
| `CONFLICT` | A resource conflict occurred, such as a file already existing (HTTP 409). |
| `FILE_TOO_LARGE` | The file exceeds the maximum allowed upload size (HTTP 413). |
| `RATE_LIMITED` | Too many requests sent in a short period (HTTP 429). |
| `SERVER_ERROR` | An internal error occurred on the Vault server (HTTP 500). |
| `NETWORK_ERROR` | No response received. Check your internet connection and `VAULT_BASE_URL`. |
| `WEBSOCKET_ERROR` | Failed to establish or maintain a WebSocket connection. |
| `STORAGE_UPLOAD_FAILED` | File failed to upload to the underlying storage (e.g., S3). |
| `PRESIGN_FAILED` | Failed to generate a temporary upload URL. |
| `REGISTER_FAILED` | File uploaded to storage but failed to register in the vault database. |
