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

Returns the available plans, each detailing aspects such as storage capacity, price, and duration.

## Current Storage Usage

### `getStorageDetails(vaultId)`

Provides a detailed breakdown of the current storage usage for the specified vault.

**Parameters:**

- `vaultId` (String): The ID of the vault.

**Example:**

```javascript
const storage = await vault.getStorageDetails('your-vault-id');
console.log(storage);
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

Returns information about the purchase, possibly including transaction details or updated subscription status.

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

### `cancelSubscription(vaultId)`

Cancels the active subscription at the end of the current billing period. The plan
remains usable until it expires.

**Parameters:**

- `vaultId` (String): The ID of the vault.

**Example:**

```javascript
const result = await vault.cancelSubscription('your-vault-id');
console.log(result);
```

**Response:**

Returns the cancellation scheduling details, including when the plan lapses.

## Upcoming Plans

An upcoming plan is queued to start once the currently active plan expires, so a
change of tier takes effect at the boundary rather than immediately.

### `createUpcomingPlan(vaultId, priceId)`

Schedules a plan to start after the current plan expires.

**Parameters:**

- `vaultId` (String): The ID of the vault.
- `priceId` (String): The price ID of the plan to queue up.

**Example:**

```javascript
const result = await vault.createUpcomingPlan('your-vault-id', 'price-id-for-plan');
console.log(result);
```

### `cancelUpcomingPlan(vaultId)`

Cancels auto-renewal for a pending upcoming plan, so the queued plan does not start.

**Parameters:**

- `vaultId` (String): The ID of the vault.

**Example:**

```javascript
const result = await vault.cancelUpcomingPlan('your-vault-id');
console.log(result);
```
