# Wallet Operations

This section covers retrieving the authenticated vault user's Twin Points wallet
summary.

## Get Wallet Info

### `getWalletInfo(vaultId)`

Retrieves the wallet summary for the authenticated user associated with the
vault. If the wallet does not exist yet, the backend creates it before returning
the summary.

**Parameters:**

- `vaultId` (String): The vault ID used to identify the authenticated vault
  context. Required.

The SDK sends `vaultId` as a URL query parameter. The server still determines
the wallet from the authenticated user, so the caller must have access to the
specified vault.

**Example:**

```javascript
try {
  const result = await vault.getWalletInfo('your-vault-id');

  console.log('Available Twin Points:', result.data.points);
  console.log('Wallet status:', result.data.status);
  console.log('Created at:', result.data.createdAt);
} catch (error) {
  console.error('Could not fetch wallet:', error.code, error.message);
}
```

**Response:**

Returns the standard response:

```javascript
{
  success: true,
  message: 'Wallet info fetched successfully',
  data: {
    points: 120,
    status: 'ACTIVE',
    createdAt: '2026-09-10T10:30:00.000Z'
  }
}
```

- `data.points` (Number): Current available Twin Points balance.
- `data.status` (String): Wallet status returned by the backend.
- `data.createdAt` (String): Wallet creation timestamp, normally an ISO-8601
  date string.

The wallet summary is read-only. Use the relevant plan or subscription methods
to purchase or manage storage plans; those do not change this wallet summary
directly.

**Errors:**

- `INVALID_PARAMETER`: `vaultId` is missing or is not a string.
- `UNAUTHORIZED`: Authentication failed or no authenticated user context exists.
- `FORBIDDEN`: The authenticated user cannot access the specified vault.
- `SERVER_ERROR`: The wallet could not be created or retrieved.

See **[Error Handling](../Error%20Handling/README.md)** for SDK error types and
other API or network errors.

## Get Transaction History

### `getTransactionHistory(vaultId, query)`

Retrieves the authenticated vault user's wallet transactions in pages, newest
first. You can optionally filter the results by transaction category.

**Parameters:**

- `vaultId` (String): The vault ID used to identify the authenticated vault
  context. Required.
- `query` (Object, optional): Pagination and filtering options. Defaults to `{}`.
  - `page` (Number): Page number, starting at `1`. Defaults to `1`.
  - `limit` (Number): Number of transactions per page. Defaults to `20`.
  - `category` (String): Optional transaction category filter. Blank values are
    ignored and non-string values are not sent.

The SDK sends query values as strings in the URL. The backend scopes transactions
to the authenticated user; it does not expose another user's wallet history.

**Example:**

```javascript
try {
  const result = await vault.getTransactionHistory('your-vault-id', {
    page: 2,
    limit: 10,
    category: 'credit',
  });

  for (const transaction of result.data.transactions) {
    console.log(transaction.transactionType, transaction.amount, transaction.createdAt);
  }

  console.log('Page:', result.data.pagination.page);
  console.log('Total transactions:', result.data.pagination.total);
} catch (error) {
  console.error('Could not fetch transaction history:', error.code, error.message);
}
```

Fetch the first page with defaults:

```javascript
const result = await vault.getTransactionHistory('your-vault-id');
```

**Response:**

Returns the standard response:

```javascript
{
  success: true,
  message: 'Transaction history fetched successfully',
  data: {
    transactions: [
      {
        id: 'transaction-id',
        userId: 'user-id',
        amount: 10,
        transactionType: 'CREDIT',
        transactionCategory: 'credit',
        createdAt: '2026-09-10T10:30:00.000Z'
      }
    ],
    pagination: {
      page: 1,
      limit: 20,
      total: 42,
      totalPages: 3
    }
  }
}
```

- `data.transactions` contains the current page, ordered by `createdAt`
  descending.
- `data.pagination.page` and `limit` describe the requested page.
- `data.pagination.total` is the number of matching transactions.
- `data.pagination.totalPages` is the number of available pages.

Transaction fields are returned by the backend and can include the transaction
ID, amount, type/category, description, metadata, and timestamps. Treat the
response as read-only history; use wallet, plan, or subscription methods for
new account actions.

**Errors:**

- `INVALID_PARAMETER`: `vaultId` is missing or not a string, or `query` is not
  an object.
- `UNAUTHORIZED`: Authentication failed or no authenticated user context exists.
- `FORBIDDEN`: The authenticated user cannot access the specified vault.
- `SERVER_ERROR`: The transaction history could not be retrieved.

See **[Error Handling](../Error%20Handling/README.md)** for SDK error types and
other API or network errors.
