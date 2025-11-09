# Sign and Broadcast Transactions Manually

## Introduction

Understanding how to **sign and broadcast transactions manually** is a fundamental skill in Web3 security and blockchain development.

### Goal

The goal of this document is to demonstrate **how to sign a transaction and broadcast it manually** By the end, you'll understand:

1. How transactions are structured
2. The signing process using private keys
3. How to broadcast signed transactions to the network

#### Tools used

- [SpotBlock Tools](https://spotblock.org/tools)
- [SpotBlock Broadcast Sepolia Demo](https://spotblock.org/broadcast-sepolia)
- [Foundry Cast (CLI)](https://book.getfoundry.sh/cast/)
- [Rabby Wallet](https://rabby.io/)
- [ethers.js library](https://docs.ethers.org/v6/)



### 1. Generate an Unsigned Transaction

Before we can sign and broadcast a transaction, we need to generate the unsigned transaction data. There are multiple ways to do this:

#### Method 1: Using Rabby Wallet [Visual interface]

Rabby Wallet provides a user-friendly interface to generate unsigned transactions:

1. For contracts that do not have a web application, it's not possible to use rabby wallet.
2. Use the SpotBlock test contract on Sepolia to generate an unsigned tx.
3. Connect your wallet
4. Pick a number and click on send transaction.
5. On rabby wallet, in "Unknown signature type", Click on view  And there you have the unsigned tx.

---

#### Method 2: Using Foundry's `cast` Command

If the contract has not web interface, the only way to interact with, is to use specific tools like cast. Most of the mixers are only accessible via this method (no UI).

Requirements:
1. Retrieve the value name of the function you want to call.
2. If the ABI (function) is not provided you need to reverse engineer the ABI, check what a function selector for more info.
3. SpotBlock contract exposes setValue() and getValue() functions. Check: https://sepolia.etherscan.io/address/0x74CAF34449834F0E0F6823a0a4B700694b5263D2#code

cast calldata will create the hex string we need to insert in our data field in our tx.
```bash
cast calldata "setValue(uint256)" 5
0x552410770000000000000000000000000000000000000000000000000000000000000005
```


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
#### Understanding the Transaction Object

- **`to`**: The recipient address (contract or EOA)
- **`data`**: The encoded function call data (in this case, calling a contract function with parameter `3`)
- **`gasLimit`**: Maximum gas units allowed (`0xa734` = 42,804 in decimal)
- **`gasPrice`**: Price per gas unit (`0x1312e2` = 1,250,018 wei)
- **`value`**: Amount of ETH to send (`0x0` = 0 ETH)
- **`nonce`**: Transaction number from sender's address (`0x8` = 8th transaction)
- **`chainId`**: Network identifier (11155111 = Sepolia testnet)
- **`type`**: Transaction type (0 = legacy, 2 = EIP-1559)


Get your nonce and gas value via:

```bash
cast nonce YOUR_ADDRESS --rpc-url https://sepolia.infura.io/v3/YOUR_INFURA_ID
cast gas-price --rpc-url https://sepolia.infura.io/v3/YOUR_INFURA_ID
```

For more check the EIP-155.

---

### 2. Sign the Transaction Using Ethers.js

Sadly, we can't sign our unsigned tx using foundry. Therefore, we will use ethers.js, a lib.

Once you have the unsigned transaction data, the next step is to sign it using a private key. Here's how to do it with ethers.js:

#### Setup

First, install ethers.js if you haven't already:

```bash
npm install ethers
```

#### Signing Script

Note the field keys need to be modified from rabby wallet because rabby is likely not using ethers.
ex:gas becomes gasLimit

Create a file `signTx.js` with the following code:

```javascript
const { ethers } = require("ethers");

// Define the transaction object
const tx = {
  to: "0x74CAF34449834F0E0F6823a0a4B700694b5263D2",
  data: "0x552410770000000000000000000000000000000000000000000000000000000000000009",
  gasLimit: 0xa734,
  gasPrice: 0x1312e2,
  value: 0x0,
  nonce: 0x9,
  chainId: 11155111, // Sepolia testnet
  type: 0, // legacy tx
};

// Create a wallet instance with your private key
// Never hardcode private keys in production!
const wallet = new ethers.Wallet("YOUR_PRIVATE_KEY_HERE");

// Sign the transaction
wallet.signTransaction(tx).then(signedTx => {
  console.log("Signed Transaction:");
  console.log(signedTx);
}).catch(err => {
  console.error("Error:", err.message);
});
```

#### Run the Script

```bash
node signTx.js
```


#### Output

The script will output a **signed transaction** in hexadecimal format that looks like:

```
0xf8aa088301312e82a73494...your_signed_transaction_hex...
```

This signed transaction contains:
- All the original transaction data
- The cryptographic signature (v, r, s values)
- Can be broadcast to the network without further modification

---


### 3 - Broadcasting

Once you have produced a signed transaction hex (either from Foundry, ethers.js, etc.), you can broadcast (send) it to the network using several approaches:

### 1. Using Foundry's `cast publish`
```bash
cast publish 0xYOUR_SIGNED_TX_HEX --rpc-url https://sepolia.infura.io/v3/YOUR_INFURA_ID
```
Where `0xYOUR_SIGNED_TX_HEX` is your full signed transaction.

### 2. Using your Web Broadcast Tool
- Go to your SpotBlock website's `/tools` page, select the appropriate network.
- Paste the signed transaction hex (ensure it starts with `0x` and is complete).
- Click 'Broadcast'.

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
You’ll get a JSON response with the transaction hash if successful.

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



Thanks reading ! Happy hacking.