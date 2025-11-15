~Was wondering what is happening behind infura calls more in details.

Ethereum exposes a **JSON-RPC API** over HTTP or WebSocket that lets clients invoke node operations remotely.

When you use Infura, you’re simply sending these same JSON payloads to Infura’s endpoints instead of your own Geth or Parity node.

## 🔹 1. Core Namespaces & Common Methods

### `eth_` – Blockchain & Transaction Control

|Method|Description|
|---|---|
|`eth_blockNumber`|Returns the latest block number|
|`eth_getBalance`|Returns ETH balance of an address|
|`eth_call`|Executes a **read-only** call (view/pure functions)|
|`eth_sendRawTransaction`|Sends a pre-signed transaction to the network|
|`eth_getTransactionReceipt`|Gets status, logs, and gas used of a tx after it's mined|
|`eth_getTransactionByHash`|Gets details of a transaction|
|`eth_getCode`|Returns the bytecode of a contract at an address|
|`eth_getStorageAt`|Reads a specific storage slot|

---

### `net_` – Network Status

|Method|Description|
|---|---|
|`net_version`|Returns the network ID (`1` = mainnet)|
|`net_listening`|Returns true if the node is listening|
|`net_peerCount`|Number of peers connected to the node|

---

### `web3_` – Client Info

|Method|Description|
|---|---|
|`web3_clientVersion`|Returns node version (e.g. Geth/vX.X.X)|
|`web3_sha3`|Computes Keccak256 hash of input data|

---

## 🔐 2. Account & Admin Namespaces

### `personal_` – Account Management

|Method|Description|
|---|---|
|`personal_listAccounts`|Lists local accounts (if any unlocked)|
|`personal_unlockAccount`|Unlocks an account for signing|
|`personal_sendTransaction`|Sends a transaction after local signing|

### `admin_` – Node Admin (only for self-hosted nodes)

|Method|Description|
|---|---|
|`admin_peers`|Lists connected peers|
|`admin_startWS` / `admin_startHTTP`|Starts a WebSocket or HTTP server interface|

---

## 🧪 3. Debugging & Tracing

### `debug_` – Low-level Execution Tracing

|Method|Description|
|---|---|
|`debug_traceTransaction`|Traces a tx and returns opcode-level execution|
|`debug_printBlock`|Prints all tx and state diffs in a block|

### `txpool_` – Mempool Inspection

|Method|Description|
|---|---|
|`txpool_status`|Number of pending and queued txs|
|`txpool_content`|Full tx details in the mempool|
