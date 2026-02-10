# Storage & Plans

This section covers functions related to querying and managing storage plans and subscriptions for your Vault.

## Check Available Plans

### `getAllPlans(vaultId)`

Retrieves a list of all available storage plans for the vault.

**Parameters:**

- `vaultId` (String): The ID of the current vault.

**Example:**

```javascript
const plans = await vault.getAllPlans('your-vault-id');
console.log(plans);
```

**Response:**

Returns an array of plan objects, each detailing aspects such as storage capacity, price, and duration.

## Current Storage Usage

### `getStorageDetails(vaultId)`

Provides a detailed breakdown of the current storage usage for the specified vault.

**Parameters:**

- `vaultId` (String): The ID of the vault.

**Example:**

```javascript
const storage = await vault.getStorageDetails('your-vault-id');
console.log(`Used: ${storage.used} / Total: ${storage.total}`);
```

## Purchase a Plan

### `buyPlan(vaultId, priceId)`

Initiates a purchase for a specific storage plan.

**Parameters:**

- `vaultId` (String): The ID of the vault for which the plan is being purchased.
- `priceId` (String): The unique identifier of the price for the plan (often from Stripe or similar service).

**Example:**

```javascript
const purchaseResult = await vault.buyPlan('your-vault-id', 'price-id-for-plan');
console.log(purchaseResult);
```

**Response:**

Returns information about the successful purchase, possibly including transaction details or updated subscription status.

## Manage Subscriptions

### `getSubscriptions(vaultId)`

Lists all active subscriptions associated with the vault.

**Parameters:**

- `vaultId` (String): The ID of the vault.

**Example:**

```javascript
const subscriptions = await vault.getSubscriptions('your-vault-id');
console.log(subscriptions);
```
