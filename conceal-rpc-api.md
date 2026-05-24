# Conceal Wallet RPC API Documentation

## Overview

**conceal-rpc** is the official JSON‑RPC 2.0 HTTP server for the Conceal (CCX) cryptocurrency. It provides a full wallet management interface – create, import, unlock, send transactions, sync with the daemon, and export keys – all through standard JSON‑RPC calls.

The server can run in one of two modes:

* **Single‑wallet mode** – manages exactly one wallet at a time.  
* **Multi‑wallet mode** – manages many wallets simultaneously, with the ability to load/unload them on demand.

All communication happens over plain HTTP. The server listens on a configurable port (default **8080**) and supports Cross‑Origin Resource Sharing (CORS) for browser‑based wallets.

---

## Transport

| Item          | Value                                          |
|---------------|------------------------------------------------|
| Endpoint      | `http://<host>:<port>/json_rpc`                |
| HTTP Method   | `POST`                                         |
| Content‑Type  | `application/json`                             |
| Protocol      | JSON‑RPC 2.0 (also accepts a bare `/` POST)    |

### Request Format (JSON‑RPC 2.0)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "methodName",
  "params": { … }
}
```

### Response Format

**Success:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { … }
}
```

**Error:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32601,
    "message": "Method not found"
  }
}
```

---

## Error Codes

### Standard JSON‑RPC errors

| Code      | Message                |
|-----------|------------------------|
| `-32700`  | Parse error            |
| `-32601`  | Method not found       |
| `-32602`  | Invalid params         |
| `-32603`  | Internal error         |

### Wallet‑specific errors (used in `error.code`)

| Code  | Constant               | Description                                      |
|-------|------------------------|--------------------------------------------------|
| `-1`  | `WALLET_LOCKED`        | Wallet is locked (single‑wallet mode)            |
| `-2`  | `WALLET_NOT_FOUND`     | No wallet exists (create/import first)           |
| `-3`  | `WALLET_ALREADY_EXISTS`| A wallet already exists                          |
| `-4`  | `INVALID_PASSWORD`     | Wrong password provided                          |
| `-5`  | `SYNC_IN_PROGRESS`     | A sync operation is already running              |
| `-6`  | `NO_WALLET_LOADED`     | No wallet loaded (multi‑wallet mode)             |

---

## Command‑Line Reference

The server binary (`conceal-rpc`) accepts the following options:

| Flag                        | Description                                                 | Default            |
|-----------------------------|-------------------------------------------------------------|--------------------|
| `--port <port>`             | RPC listen port                                             | `8080`             |
| `--data-dir <path>`         | Wallet data directory                                       | `./wallet_data`    |
| `--daemon-host <host>`      | Daemon host                                                 | `127.0.0.1`        |
| `--daemon-port <port>`      | Daemon RPC port                                             | `16000`            |
| `--cors <domain>`           | CORS domain (e.g., `*`)                                     | (none)             |
| `--testnet`                 | Use testnet                                                 | `false`            |
| `--generate-wallet`         | Create a new wallet on startup                              | `false`            |
| `--import-keys <sp:view>`   | Import wallet from `spendkey:viewkey` hex pair              |                    |
| `--unlock`                  | Auto‑unlock existing wallet                                 | `false`            |
| `--password <pwd>`          | Password (otherwise prompted)                               |                    |
| `--offline`                 | Start without daemon connection (no sync)                   | `false`            |
| `--multi-wallet [<dir>]`    | Multi‑wallet mode (optional wallets directory)              | `./wallet_data/wallets` |

---

## Method Index

Methods are grouped by functional category. Each method name links to its detailed description.

### Wallet Management
| Method | Single‑Wallet | Multi‑Wallet |
|--------|:---:|:---:|
| [`createWallet`](#createwallet)                         | ✅ | ✅ |
| [`importWallet`](#importwallet)                         | ✅ | ✅ |
| [`unlockWallet`](#unlockwallet-single-mode)             | ✅ | ❌* |
| [`lockWallet`](#lockwallet-single-mode)                 | ✅ | ❌* |
| [`setActiveWallet`](#setactivewallet)                   | ✅ | ❌ |
| [`loadWallet`](#loadwallet)                             | ❌ | ✅ |
| [`unloadWallet`](#unloadwallet)                         | ❌ | ✅ |
| [`deleteWallet`](#deletewallet)                         | ❌ | ✅ |
| [`listWallets`](#listwallets)                           | ✅ | ✅ |
| [`changePassword`](#changepassword)                     | ❌ | ✅ |
| [`importFromStateFile`](#importfromstatefile)           | ❌ | ✅ |

*In multi‑wallet mode the legacy names `unlockWallet` and `lockWallet` are aliased to `loadWallet` and `unloadWallet`.*

### Querying
| Method | Single‑Wallet | Multi‑Wallet |
|--------|:---:|:---:|
| [`getStatus`](#getstatus)                               | ✅ | ✅ |
| [`getBalance`](#getbalance)                             | ✅ | ✅ |
| [`getAddress`](#getaddress)                             | ✅ | ✅ |
| [`getOutputs`](#getoutputs)                             | ✅ | ✅ |
| [`getTransactions`](#gettransactions)                   | ✅ | ✅ |

### Transactions
| Method | Single‑Wallet | Multi‑Wallet |
|--------|:---:|:---:|
| [`sendTransaction`](#sendtransaction)                   | ✅ | ✅ |
| [`sendDeposit`](#senddeposit)                           | ✅ | ✅ |
| [`sendWithdrawal`](#sendwithdrawal)                     | ✅ | ✅ |

### Sync & Export
| Method | Single‑Wallet | Multi‑Wallet |
|--------|:---:|:---:|
| [`syncNow`](#syncnow)                                   | ✅ | ✅ |
| [`exportKeys`](#exportkeys)                             | ✅ | ✅ |
| [`exportState`](#exportstate)                           | ✅ | ✅ |

---

## Method Details

---

### `createWallet`

Create a brand new wallet with a fresh key pair.

**Availability:** Single‑wallet & Multi‑wallet

**Parameters**

| Name       | Type   | Required | Description                         |
|------------|--------|:--------:|-------------------------------------|
| `wallet_id`| string | multi‑only | Unique wallet identifier             |
| `password` | string | yes      | Password (min 8 characters)         |

**Single‑wallet mode:**  
A wallet must **not** already exist (error `-3`). The wallet is automatically unlocked after creation.

**Multi‑wallet mode:**  
The `wallet_id` must be unique. The new wallet is **not** automatically loaded.

**Response**

```json
{
  "address": "ccx7…",
  "message": "Wallet created successfully"
}
```

Multi‑wallet additionally returns:
```json
{
  "wallet_id": "mywallet",
  "state_file": "/path/to/wallet.state"
}
```

**Possible errors**

| Code | Condition                                 |
|------|-------------------------------------------|
| `-2` | Missing `password` parameter              |
| `-3` | Wallet already exists (single mode)       |
|      | `wallet_id` already exists (multi mode)   |
| `-3` | Password too short (< 8 chars)            |
| `-3` | Internal key generation failure           |

---

### `importWallet`

Import a wallet from existing spend and view secret keys.

**Availability:** Single‑wallet & Multi‑wallet

**Parameters**

| Name       | Type   | Required | Description                              |
|------------|--------|:--------:|------------------------------------------|
| `wallet_id`| string | multi‑only | Unique wallet identifier                  |
| `viewKey`  | string | yes      | View secret key (hex)                    |
| `spendKey` | string | yes      | Spend secret key (hex)                   |
| `password` | string | yes      | Password to encrypt the local key file   |

**Single‑wallet mode:**  
A wallet must not already exist. The wallet is unlocked after import.

**Multi‑wallet mode:**  
`wallet_id` must be unique; wallet is not loaded automatically.

**Response**

```json
{
  "address": "ccx7…",
  "message": "Wallet imported successfully"
}
```

Multi‑wallet adds `wallet_id` and `state_file`.

**Possible errors**

| Code | Condition                              |
|------|----------------------------------------|
| `-2` | Missing required parameters            |
| `-3` | Wallet already exists                  |
| `-3` | Invalid key hex or key pair            |
| `-3` | Internal failure                      |

---

### `unlockWallet` (Single‑mode)

Decrypt and unlock an existing wallet, starting background sync.

**Availability:** Single‑wallet mode only (in multi‑wallet mode it’s an alias for `loadWallet`)

**Parameters**

| Name       | Type   | Required | Description      |
|------------|--------|:--------:|------------------|
| `password` | string | yes      | Wallet password  |

**Response**

```json
{
  "address": "ccx7…",
  "balance": 100000,
  "message": "Wallet unlocked"
}
```

**Possible errors**

| Code | Condition                    |
|------|------------------------------|
| `-2` | Missing password             |
| `-2` | No wallet found (create/import first) |
| `-4` | Invalid password             |

---

### `lockWallet` (Single‑mode)

Lock the wallet, wiping keys from memory and stopping sync.

**Availability:** Single‑wallet mode only (multi‑wallet alias for `unloadWallet`)

**Parameters:** None

**Response**

```json
{
  "message": "Wallet locked"
}
```

---

### `setActiveWallet`

In single‑wallet mode, switch the active wallet name (used for multi‑wallet‑file single‑mode). This does **not** unlock the wallet – call `unlockWallet` afterwards.

**Availability:** Single‑wallet mode

**Parameters**

| Name         | Type   | Required | Description     |
|--------------|--------|:--------:|-----------------|
| `walletName` | string | yes      | Wallet filename (without `.keys` extension) |

**Response**

```json
{
  "activeWallet": "mywallet",
  "hasWallet": 1,
  "message": "Active wallet set. Use unlockWallet to open."
}
```

---

### `loadWallet`

In multi‑wallet mode, decrypt and load a wallet into memory, making it the active wallet.

**Availability:** Multi‑wallet mode

**Parameters**

| Name       | Type   | Required | Description      |
|------------|--------|:--------:|------------------|
| `wallet_id`| string | yes      | Wallet identifier |
| `password` | string | yes      | Wallet password   |

**Response**

```json
{
  "wallet_id": "mywallet",
  "address": "ccx7…",
  "balance": 100000,
  "height": 123456
}
```

**Possible errors**

| Code | Condition                   |
|------|-----------------------------|
| `-2` | Missing parameters          |
| `-4` | Wallet not found or wrong password |

---

### `unloadWallet`

Unload the currently active wallet, saving its state and stopping background sync.

**Availability:** Multi‑wallet mode

**Parameters:** None

**Response**

```json
{
  "success": true
}
```

**Errors:** `-6` if no wallet is loaded.

---

### `deleteWallet`

Permanently delete a wallet’s state file and index entry. The wallet must **not** be the active wallet (unload it first).

**Availability:** Multi‑wallet mode

**Parameters**

| Name       | Type   | Required | Description      |
|------------|--------|:--------:|------------------|
| `wallet_id`| string | yes      | Wallet identifier |
| `password` | string | yes      | Wallet password (to verify ownership) |

**Response**

```json
{
  "success": true,
  "deleted": "mywallet"
}
```

**Possible errors**

| Code | Condition                              |
|------|----------------------------------------|
| `-2` | Missing parameters                     |
| `-6` | Wallet is currently loaded              |
| `-4` | Wrong password                         |
| `-2` | Wallet not found                       |

---

### `listWallets`

List all known wallets and their basic info. In multi‑wallet mode the active wallet shows its real balance.

**Availability:** Both modes

**Parameters:** None

**Response**

```json
{
  "wallets": [
    {
      "name": "mywallet",
      "address": "ccx7…",
      "balance": 100000,
      "loaded": true
    }
  ]
}
```

Single‑wallet mode returns only filenames without balances.

---

### `changePassword`

Change the encryption password of the currently loaded wallet (multi‑wallet mode).

**Availability:** Multi‑wallet mode

**Parameters**

| Name           | Type   | Required | Description        |
|----------------|--------|:--------:|--------------------|
| `old_password` | string | yes      | Current password   |
| `new_password` | string | yes      | New password       |

**Response**

```json
{
  "success": true
}
```

**Errors:** `-6` (no wallet loaded), `-4` (old password wrong)

---

### `importFromStateFile`

Import a wallet using an existing `.state` file (full backup) and register it under a new `wallet_id`.

**Availability:** Multi‑wallet mode

**Parameters**

| Name        | Type   | Required | Description                          |
|-------------|--------|:--------:|--------------------------------------|
| `wallet_id` | string | yes      | New wallet identifier                |
| `file_path` | string | yes      | Path to the source `.state` file     |
| `password`  | string | yes      | Password used to encrypt the backup  |

**Response**

```json
{
  "wallet_id": "imported1",
  "address": "ccx7…",
  "state_file": "/data/imported1.state"
}
```

---

## Querying Methods

### `getStatus`

Get the current status of the (active) wallet, including sync progress.

**Availability:** Both modes

**Parameters:** None (multi‑wallet automatically uses the active wallet)

**Response**

```json
{
  "locked": 0,
  "synced": 1,
  "blockHeight": 123456,
  "balance": 500000,
  "unlockedBalance": 300000,
  "outputCount": 12,
  "transactionCount": 45,
  "syncProgress": {                       // Only present if a sync is active
    "phase": 2,
    "totalBlocks": 150000,
    "processedBlocks": 80000,
    "candidatesFound": 200,
    "ownedOutputs": 5,
    "currentHeight": 150000,
    "errorMessage": ""                     // If an error occurred
  }
}
```

**Errors:** `-6` in multi‑wallet mode if no wallet loaded.

---

### `getBalance`

Return the total and unlocked balance.

**Availability:** Both modes

**Parameters:** None

**Response**

```json
{
  "balance": 500000,
  "unlockedBalance": 300000
}
```

**Errors:** `-1` (single) or `-6` (multi) if wallet locked/not loaded.

---

### `getAddress`

Return the wallet’s public address.

**Availability:** Both modes

**Response**

```json
{
  "address": "ccx7…"
}
```

**Errors:** `-1`/`-6`

---

### `getOutputs`

List owned outputs (UTXOs).

**Availability:** Both modes

**Parameters**

| Name          | Type    | Default | Description                            |
|---------------|---------|:-------:|----------------------------------------|
| `unspentOnly` | integer | `1`     | `1` to return only unspent outputs     |

**Response**

```json
{
  "outputs": [
    {
      "blockHeight": 100000,
      "txHash": "abcdef...",
      "amount": 10000,
      "outputIndex": 0,
      "outputKey": "0123...",
      "txPublicKey": "4567...",
      "spent": 0,
      "isDeposit": 0,
      "term": 0
    }
  ],
  "count": 1
}
```

**Errors:** `-1`/`-6`

---

### `getTransactions`

Retrieve transaction history with pagination.

**Availability:** Both modes

**Parameters**

| Name     | Type   | Default | Description           |
|----------|--------|:-------:|-----------------------|
| `offset` | integer| `0`     | Starting index        |
| `limit`  | integer| `50`    | Max records to return |

**Response**

```json
{
  "transactions": [
    {
      "txHash": "abcdef...",
      "blockHeight": 100000,
      "timestamp": 1690000000,
      "fee": 100,
      "totalSent": 0,
      "totalReceived": 50000,
      "type": 1,                // 0=incoming, 1=outgoing, 2=deposit, 3=withdrawal
      "confirmed": 1,
      "paymentId": ""
    }
  ]
}
```

**Errors:** `-1`/`-6`

---

## Transaction Methods

### `sendTransaction`

Build and relay a standard transfer.

**Availability:** Both modes

**Parameters**

| Name        | Type   | Required | Description                                      |
|-------------|--------|:--------:|--------------------------------------------------|
| `address`   | string | yes      | Destination Conceal address                       |
| `amount`    | integer| yes      | Amount in atomic units (1 CCX = 10^6 units)       |
| `paymentId` | string | no       | Optional payment ID (hex)                         |
| `mixin`     | integer| no       | Ring size (currently fixed by protocol)           |
| `fee`       | integer| no       | Transaction fee in atomic units                   |

**Response**

```json
{
  "txHash": "abcdef...",
  "fee": 100
}
```

**Errors:** `-1`/`-6` if wallet locked/not loaded; `-2` if `address` or `amount` missing; `-3` on failure (error message in `error.message`).

> **Note:** The transaction builder is currently a placeholder and returns an error. Full functionality will be enabled in a future release.

---

### `sendDeposit`

Create a time‑locked deposit (staking) transaction.

**Availability:** Both modes

**Parameters**

| Name     | Type   | Required | Description                                  |
|----------|--------|:--------:|----------------------------------------------|
| `amount` | integer| yes      | Amount to deposit (atomic units)             |
| `term`   | integer| yes      | Lock period in blocks                        |
| `fee`    | integer| no       | Transaction fee                              |

**Response**

```json
{
  "txHash": "abcdef...",
  "fee": 100
}
```

**Errors:** `-1`/`-6`; `-2` if parameters missing.

> **Note:** Deposit builder is currently a placeholder.

---

### `sendWithdrawal`

Withdraw from an existing time‑locked deposit.

**Availability:** Both modes

**Parameters**

| Name        | Type   | Required | Description                          |
|-------------|--------|:--------:|--------------------------------------|
| `amount`    | integer| yes      | Amount to withdraw (atomic units)    |
| `depositId` | integer| yes      | ID of the deposit output             |
| `fee`       | integer| no       | Transaction fee                      |

**Response**

```json
{
  "txHash": "abcdef...",
  "fee": 100
}
```

**Errors:** `-1`/`-6`; `-2` if parameters missing.

> **Note:** Withdrawal builder is currently a placeholder.

---

## Sync & Export

### `syncNow`

Trigger an immediate rescan for new outputs. In multi‑wallet mode sync is continuous; this call has no effect but returns success.

**Availability:** Both modes

**Response**

```json
{
  "message": "Sync triggered"
}
```

**Errors:** `-1`/`-6` if wallet locked/not loaded.

---

### `exportKeys`

Export the wallet’s secret keys.

**Availability:** Both modes

**Response**

```json
{
  "keys": "spendkeyhex:viewkeyhex",
  "warning": "Keep these keys secure. Anyone with these keys can access your funds."
}
```

**Errors:** `-1`/`-6`

---

### `exportState`

Export the path to the wallet’s current state file (`.state`). The file can be copied for backup or migration.

**Availability:** Both modes

**Response**

```json
{
  "path": "/home/user/wallet_data/mywallet.state",
  "message": "State file path. Copy this file to backup or migrate your wallet."
}
```

**Errors:** `-1`/`-6`

---

## Usage Examples

All examples assume the server is running on `localhost:8080`.

**Create and unlock a new wallet (single mode):**

```bash
curl -s -X POST http://localhost:8080/json_rpc \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0", "id":1, "method":"createWallet",
    "params": {"password":"strongpassword123"}
  }'
```

**Get wallet status:**

```bash
curl -s -X POST http://localhost:8080/json_rpc \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0", "id":1, "method":"getStatus"}'
```

**Multi‑wallet: create a wallet, then load it:**

```bash
# Create
curl -s -X POST http://localhost:8080/json_rpc \
  -d '{"jsonrpc":"2.0","id":1,"method":"createWallet","params":{"wallet_id":"alice","password":"pw12345678"}}'

# Load
curl -s -X POST http://localhost:8080/json_rpc \
  -d '{"jsonrpc":"2.0","id":2,"method":"loadWallet","params":{"wallet_id":"alice","password":"pw12345678"}}'
```

**List all wallets (multi mode):**

```bash
curl -s -X POST http://localhost:8080/json_rpc \
  -d '{"jsonrpc":"2.0","id":1,"method":"listWallets"}'
```

**Export keys (active wallet):**

```bash
curl -s -X POST http://localhost:8080/json_rpc \
  -d '{"jsonrpc":"2.0","id":1,"method":"exportKeys"}'
```

---

## Data Types & Units

* All amounts are in **atomic units** (1 CCX = 1 000 000 atomic units).
* Timestamps are Unix seconds.
* Block heights are unsigned 32‑bit integers.
