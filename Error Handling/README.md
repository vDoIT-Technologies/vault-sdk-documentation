# Error Handling

The Vault SDK provides a robust error-handling mechanism to help you identify and resolve issues quickly. All errors thrown by the SDK are categorized into two main types: `ValidationError` and `VaultError`.

## Error Types

### `ValidationError`

Thrown when the parameters passed to an SDK method fail validation (e.g., a required parameter is missing or has the wrong type).

**Properties:**
- `message`: A descriptive message indicating what failed.
- `code`: Always `"INVALID_PARAMETER"`.
- `param`: The name of the parameter that failed validation.
- `expectedType`: The type the parameter was expected to be (e.g. `"string"`, `"boolean"`, `"array"`).
- `operation`: The name of the method where the error occurred.

### `VaultError`

Thrown for API errors, network issues, or internal SDK failures.

**Properties:**
- `message`: Descriptive error message.
- `code`: A unique string identifier for the error (see [Error Codes](#error-codes)).
- `status`: The HTTP status code (if applicable).
- `operation`: The name of the method or endpoint where the error occurred.
- `data`: The raw error response data from the server (if available).

> **Note:** `code: "INVALID_PARAMETER"` is not exclusive to `ValidationError`.
> `uploadFile()` throws a `VaultError` with that code when the file itself is
> unusable — for example a zero-byte file, a bare `Buffer` with no name, or an
> object with no readable content. Branch on `error.code` rather than on the error
> class when you care about the specific cause.

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

### Configuration & parameters

| Code | Description |
|------|-------------|
| `MISSING_CONFIG` | Required configuration parameter not provided. Thrown by the constructor when any of `VAULT_ACCESS_KEY`, `VAULT_SECRET_KEY`, `VAULT_CLIENT_API_KEY`, or `VAULT_BASE_URL` is missing. |
| `INVALID_PARAMETER` | Method parameter failed validation, or the supplied file was unusable (empty, unnamed, or malformed). |

### HTTP responses from the server

| Code | Description |
|------|-------------|
| `BAD_REQUEST` | The server rejected the request (HTTP 400). Check your payload. |
| `UNAUTHORIZED` | Authentication failed (HTTP 401). Check your Access Key and Secret Key. |
| `FORBIDDEN` | Your API key does not have permission for this operation (HTTP 403). |
| `NOT_FOUND` | The requested resource (vault, file, folder) does not exist (HTTP 404). |
| `CONFLICT` | A resource conflict occurred, such as a file already existing (HTTP 409). |
| `FILE_TOO_LARGE` | The file exceeds the maximum allowed upload size (HTTP 413, or the SDK's own 10 GB check). |
| `RATE_LIMITED` | Too many requests sent in a short period (HTTP 429). |
| `SERVER_ERROR` | An internal error occurred on the Vault server (HTTP 500). |
| `BAD_GATEWAY` | The server is temporarily unavailable (HTTP 502). Retry shortly. |
| `SERVICE_UNAVAILABLE` | The service is temporarily unavailable (HTTP 503). Retry shortly. |
| `UNKNOWN_ERROR` | The server returned a status the SDK has no specific mapping for. Inspect `error.status` and `error.data`. |

### Transport

| Code | Description |
|------|-------------|
| `NETWORK_ERROR` | No response received. Check your internet connection and `VAULT_BASE_URL`. |
| `REQUEST_SETUP_ERROR` | The request could not be constructed or dispatched at all. |

### Upload pipeline

| Code | Description |
|------|-------------|
| `FILE_READ_FAILED` | The file could not be read from the given path. |
| `PRESIGN_FAILED` | Failed to generate a temporary upload URL, or the server returned no URL. |
| `STORAGE_UPLOAD_FAILED` | File failed to upload to the underlying storage (e.g., S3). |
| `REGISTER_FAILED` | File uploaded to storage but failed to register in the vault database. |
| `UPLOAD_FAILED` | Fallback code on a failed entry in the `uploadFiles()` result array when the underlying error carried no code. |

### Fallback

| Code | Description |
|------|-------------|
| `VAULT_ERROR` | Default code on a `VaultError` constructed without a more specific one. |

---

## Errors from `uploadFiles()`

`uploadFiles()` is the one method that does **not** throw on failure. Because each
file is uploaded independently, per-file errors are reported in the returned array
instead, so a single bad file does not lose the successful uploads:

```javascript
const results = await vault.uploadFiles(files, 'your-vault-id');

const failed = results.filter((result) => result.status === 'failed');
for (const failure of failed) {
  console.error(`${failure.fileName}: [${failure.code}] ${failure.error}`);
}
```

A `ValidationError` is still thrown up front if `files` is not a non-empty array or
`vaultId` is missing.
