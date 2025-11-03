# Why Maintaining Privacy on Chain Matters in Bug Bounty Hunting

TLDR:<br>
**Threats you're exposed to:**
- 🧠 **RPC providers** (Infura, Alchemy): correlate IPs, wallets, patterns
- 🛰️ **SIGINT operators** (nation-states, adversarial L1s): network metadata
- 🔍 **Other bug bounty hunters**: front-running ideas, stealing exploits
- 🏛️ **Projects themselves**: patch silently after detection
- 🦹 **Physical thieves**: if identity/strategy gets exposed

**Solutions that reduce your footprint:**
- 🧅 Route traffic through **Tor**
- 🔐 Use **private / self-hosted RPCs**
- 🧱 Maintain a **full local node** or fork with `anvil`
- 👤 Practice strong **OPSEC** (burner wallets, secure devices)
- 🧬 Use **local forked chains** for simulation
- 🔄 Rotate wallets and **obfuscate behavior patterns**
- 💻 Use **hardened OS/VMs** for research
  
<TLDR>



---
More you will progress in bug bounty and more regrets you will have to not have consider privacy from day one.

# Introduction : Monero VS Ethereum 

We all know that Monero embed specific components that increase the privacy, this includes bulletproofs, RingCTs, stealth addresses, Dandelion++.

To sum up Monero features and the associated risks:

| Risks:                                                  | Mitigated By XMR features:     |
| ------------------------------------------------------- | ------------------------------ |
| 👁️‍🗨️ Chain analysis to trace sender/receiver         | RingCT + Stealth addresses     |
| 💸 Value correlation to de-anonymize usage              | Bulletproof++ + RingCT         |
| 🕵️ IP tracking to deanonymize sender                   | Dandelion++, Tor/I2P           |
| 🧠 Re-identification from behavior (KYC, address reuse) | Stealth addresses + Good OPSEC |
| 🧮 Amount analysis via known UTXO sizes                 | Bulletproof++ (range proofs)   |

<br>

Monero handles **half the battle** by protecting your transactions on-chain and over the network. But the other **50% is your OPSEC**. 

# Problematic: The challenge for privacy on Ethereum

Ethereum, by contrast, has **zero built-in privacy**.

Every transaction is public, permanent, and linkable unless you add external privacy layers. Here is a graph of a casual user of ethereum.

<img width="842" height="862" alt="Ethereum usage drawio (1)" src="https://github.com/user-attachments/assets/39d8cf7f-dcae-48d3-872b-33281974a0db" />

Based on the diagram we understand the direct challenges for the privacy are:
- **Metamask / browser wallets**: leaks browser info, locale, OS, screen res, timestamps
- **RPC over HTTP**: no encryption = SIGINT or TLS fingerprinting
- **Infura & friends**: IP-to-wallet mapping, profiling, behavior clustering
- **Frontend dApps**: inject JS trackers, log tx intent, and wallet connections

Infura can build machine learning models to track suspicious behavior: early exploit simulation, frontrunning patterns, or anomalous opcode usage.

---

**Here is something I saw, that you should absolutely avoid !**
### Case study: The Trap of Offline Signing -  You’re Still Leaking Metadata

Sending ETH privately is relatively straightforward. 

You can use an RPC provider like Infura, provided you’ve registered using an anonymous account (e.g. with an anonymous email address).
```bash
#How to Send ETH Privately Using Tor and Curl
#Solution TOR 

curl --socks5-hostname 127.0.0.1:9050 \
  https://mainnet.infura.io/v3/YOUR_PROJECT_ID \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "eth_sendRawTransaction",
    "params": [
      "0xf86b808504a817c80082520894d46e8dd67c5d32be8058bb8eb970870f0724456788016345785d8a00008025a0fc..."
    ],
    "id": 1
  }'
```


But what if you want to interact with contracts? It gets trickier—especially when dApps abstract away complexity with multicalls.

Scenario: You build a tx with MetaMask, sign it offline, then broadcast it anonymously via Tor. 

<img width="801" height="638" alt="scenario eth drawio" src="https://github.com/user-attachments/assets/b7c92310-18aa-497a-a5e0-a57fd1cf4770" />

This approach is flawed:
- Nonce, gas fee, contract data → fetched during tx prep via **RPC on real IP**
- Later broadcasting the same tx (even via Tor) allows **easy correlation**

Pro tips: Use a throwaway wallet, route traffic through TOR, and connect to a custom RPC provider when interacting with dApps. This lets you safely observe how complex transactions are constructed — helping you reverse-engineer smart contract behavior before crafting your own tx using `web3.js` or `ethers.js`.

```javascript
//Create burner wallet for each Bug Bounty audit
//Solution: burner wallets
const wallet = ethers.Wallet.createRandom();
```

---
Here are some solutions for bug bounty that are easy to set up - limiting the exposure.
# Good practices for Research

**Solution 1 :** Clone a local blockchain using a private RPC provider with a burner account.  [Easy to set up]
```bash
#Solution: private RPC provider + TOR + no logs
torsocks anvil \
  --fork https://cloudflare-eth.com \
  --fork-block-number 19000000 \
  --no-storage-caching \
  --silent
```

Threat mitigated: Address Tracking - RPC Provider Log Correlation (no api key)

---

**Solution 2 :** Private Fork Networks with Custom Snapshots  [Easy to set up]
This is also handy to resume your work later on.

```bash
#Solution: avoid excessive RPC calls.
anvil --dump-state snapshot.json
anvil --state snapshot.json
```

Threat mitigated: RPC Provider Log Correlation

---

**Solution 3:** Custom Contract Mirrors for Isolated Testing [Easy to set up]
Instead of forking mainnet, deploy the contracts on a local chain. This is very powerful as you're performing no RPC call at ALL.

```bash
1 - go on github and clone the repo with the existing contract
2 - use anvil --silent --block-time 0 -> deploy eth blockchain
3 - anvil &
forge create src/Target.sol:TargetContract \
  --rpc-url http://localhost:8545 \
  --constructor-args "0xFeed" "0xToken" -> deploy on your local chain
```

Threat mitigated:  RPC log correlation (no forking = zero RPC calls) - External service reliance (fully offline testing)

---

**Solution 4:** Use your own full node for testing [Expensive to set up]

```bash
# Clone Geth
git clone https://github.com/ethereum/go-ethereum.git
cd go-ethereum
make geth

# Initialize data dir (optional custom genesis for testing)
./build/bin/geth init genesis.json --datadir $HOME/.ethereum

# Start your full node (mainnet or testnet)
./build/bin/geth \
  --datadir $HOME/.ethereum \
  --http \
  --http.addr 127.0.0.1 \
  --http.port 8545 \
  --http.api "eth,net,web3,debug,txpool" \
  --http.corsdomain "*" \
  --syncmode full \
  --cache 4096 \
  --ipcdisable \
  --allow-insecure-unlock
```
You can now:
- Replay txs
- Analyze trace logs
- Simulate state rewinds
- Do storage reads without external traces

---

**Solution 5:** Reporting strategy

**THIS IS VERY IMPORTANT FOR LIVE CRITICAL FINDINGS**

```bash
- Don’t reveal research address or fund origin
- Consider:
    - pgp-encrypted messages
    - burner emails
    - forwardable eth wallets
    - submitting via VPN/Tor
    - non-KYC-based reward address (e.g. Tornado-washed)
```

Threats mitigated: Deanonymization via address clustering - Reprisal or litigation risks from known identity

---


**Bonus Solution 6:** Isolated environment via containers
[Medium to set up]

Run all your tooling and test chains (e.g., Anvil, Geth, Foundry) inside an isolated environment — a **dedicated VM**, **Qubes OS AppVM**, or **Docker container**. Keep I/O under strict control.

```bash
Resource: trailofbits/eth-security-toolbox

docker run -it --rm \
  --network none \
  -v $PWD:/work \
  foundry \
  bash

# Inside container:
anvil --silent &
forge test
```

Threats mitigated: Tooling telemetry (e.g., Forge/Ganache/Hardhat stats, auto-update pings) + Accidental internet exposure (e.g., dependency downloads revealing research interest)

Pro tips: You can integrate with Tor inside the container if needed, or **block all egress** entirely with `--network none`.

---
Thank you reading, star this project for further tips, I'm posting quite often !

