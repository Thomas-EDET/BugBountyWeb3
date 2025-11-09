# Sign and Broadcast Transactions Manually

## Introduction

Understanding how to **sign and broadcast transactions manually** allows the creation of air-gapped setups, cold storage, or avoiding paid hardware wallets like ledger.

## Goal

By the end of this guide, you’ll understand:

1. How Ethereum transactions are structured
2. How to sign a transaction using private keys
3. How to broadcast a signed transaction to the network


## Tools Used

- [SpotBlock Tools](https://spotblock.org/tools)
- [SpotBlock Broadcast Sepolia Demo](https://spotblock.org/broadcast-sepolia)
- [Foundry Cast (CLI)](https://book.getfoundry.sh/cast/)
- [Rabby Wallet](https://rabby.io/)
- [ethers.js library](https://docs.ethers.org/v6/)


## 1. Generate an Unsigned Transaction

You can generate unsigned transactions using web wallets or CLI tools.

### A. Using Rabby Wallet (UI)

1. If using a contract with no HTML frontend, you can't use the wallet UI.
2. Use the SpotBlock test contract on Sepolia as an example.
3. Connect your wallet, choose a number, and click "Send Transaction."
4. In Rabby Wallet, under “Unknown signature type,” click “View” to see the unsigned transaction.

<img width="1162" height="786" alt="github-raw-tx" src="https://github.com/user-attachments/assets/fc055e47-cae8-427b-9591-98fa6fc14963" />

---

### B. Using Foundry's `cast` (CLI)

For CLI and scripting (common for DeFi/mixer contracts):

1. Retrieve the function name (from ABI or reverse engineer).
2. Use [4byte.directory](https://www.4byte.directory/) if ABI is not accessible.
3. Example from the SpotBlock contract:
    - ABI: `setValue(uint256)` and `getValue()`
    - Etherscan: [Code link](https://sepolia.etherscan.io/address/0x74CAF34449834F0E0F6823a0a4B700694b5263D2#code)

To create the calldata:
```bash
cast calldata "setValue(uint256)" 5
# Output:
# 0x552410770000000000000000000000000000000000000000000000000000000000000005
```

Unsigned transaction example:
```json
{
  "chainId": 11155111,
  "data": "0x552410770000000000000000000000000000000000000000000000000000000000000005",
  "from": "0x8ebb3c75f4784d00b38ecb050e5157fab6ca0561",
  "gas": "0xa734",
  "gasPrice": "0x1312dd",
  "nonce": "0x9",
  "to": "0x74CAF34449834F0E0F6823a0a4B700694b5263D2"
}
```

**Transaction Object Fields:**
- `to`: recipient address (contract or EOA)
- `data`: encoded calldata (function + params)
- `gasLimit`: max gas allowed (`0xa734` = 42,804)
- `gasPrice`: cost per gas unit (`0x1312dd`)
- `value`: amount in ETH (typically `0x0` unless transferring ETH)
- `nonce`: sender’s tx count (`0x9` is 9th tx)
- `chainId`: network ID (`11155111` for Sepolia testnet)
- `type`: `0` (legacy) or `2` (EIP-1559)

**Get Nonce and Gas Values:**
```bash
cast nonce YOUR_ADDRESS --rpc-url https://sepolia.infura.io/v3/YOUR_INFURA_ID
cast gas-price --rpc-url https://sepolia.infura.io/v3/YOUR_INFURA_ID
```

_Read more: [EIP-155](https://eips.ethereum.org/EIPS/eip-155)_

---

## 2. Sign the Transaction Using ethers.js

> ⚠️ **Foundry does not currently support direct transaction signing from JSON, so use ethers.js.**

### Setup

Install ethers.js:
```bash
npm install ethers
```

### Signing Script

**Note:** Some field names need adjustment (e.g., `gas` ➔ `gasLimit`). This is because rabby wallet don't use ethers directly.

Create a file `signTx.js`:
```js
const { ethers } = require("ethers");

const tx = {
  to: "0x74CAF34449834F0E0F6823a0a4B700694b5263D2",
  data: "0x552410770000000000000000000000000000000000000000000000000000000000000005",
  gasLimit: 0xa734,
  gasPrice: 0x1312e2,
  value: 0x0,
  nonce: 0x9,
  chainId: 11155111, // Sepolia testnet
  type: 0, // legacy tx
};

const wallet = new ethers.Wallet("YOUR_PRIVATE_KEY_HERE");

wallet.signTransaction(tx).then(signedTx => {
  console.log("Signed Transaction:");
  console.log(signedTx);
}).catch(err => {
  console.error("Error:", err.message);
});
```

Run your script:
```bash
node signTx.js
```

**Sample output:**
```
0xf8aa088301312e82a73494...your_signed_transaction_hex...
```

The result is a **signed transaction** ready for broadcast.


## 3. Broadcasting

Once you have a signed transaction hex, broadcast it with one of the following methods:

### 1. Using Foundry's `cast publish`
```bash
cast publish 0xYOUR_SIGNED_TX_HEX --rpc-url https://sepolia.infura.io/v3/YOUR_INFURA_ID
```

### 2. Using the SpotBlock Web Tool

- Go to your [SpotBlock Tools](https://spotblock.org/tools) page and select the network.
- Paste the signed tx hex (starts with `0x`).
- Click 'Broadcast'.

<img width="543" height="502" alt="broadcast" src="https://github.com/user-attachments/assets/1b2d5d80-22f4-474e-a1df-33ecf486fb74" />

### 3. With `curl` and JSON-RPC
```bash
curl -X POST https://sepolia.infura.io/v3/YOUR_INFURA_ID \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "eth_sendRawTransaction",
    "params": ["0xYOUR_SIGNED_TX_HEX"]
  }'
```
You'll get a JSON response with the transaction hash if successful.

### 4. With `ethers.js` (Node.js)
```js
const { ethers } = require("ethers");
const provider = new ethers.JsonRpcProvider("https://sepolia.infura.io/v3/YOUR_INFURA_ID");

async function main() {
  const txHash = await provider.broadcastTransaction("0xYOUR_SIGNED_TX_HEX");
  console.log("Tx Hash:", txHash.hash);
}
main();
```

---

> After broadcasting, you can view your transaction on Etherscan or a block explorer for your chain using the returned transaction hash. 

You have sucessfully sign and broadcast manually your tx !

<img width="537" height="641" alt="Screenshot from 2025-11-09 20-50-27" src="https://github.com/user-attachments/assets/2db6c36d-158d-4291-93f1-49c01813b28a" />

---

Thanks for reading! Happy hacking.
