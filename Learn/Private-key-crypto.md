# Get wallet  private key by disclosing k


| Term       | Role                         |
| ---------- | ---------------------------- |
| ECDSA      | Signature algorithm          |
| `k`        | Per-signature random number  |
| `r, s`     | The actual signature pair    |
| `v`        | Helper to recover public key |
| Randomness | Critical for security        |
| Curve      | secp256k1                    |
If `k` (the random number used in the signature) is ever reused or predictable:
- An attacker can compute your **private key**.
### **How is `k` Generated in Ethereum?**

Ethereum clients use:
- **Secure RNG (Random Number Generator)** from the OS or cryptographic libraries.
- Signing libraries (like OpenSSL or libsecp256k1) implement this safely **if** used correctly.

But some wallets, especially poorly written ones or hacked implementations, may:
- Reuse `k`
- Use weak RNG (e.g., deterministic or PRNG without entropy)

**in real life**, attackers often get access to ECDSA signatures **directly from the blockchain**.

When you send a transaction:
- You **sign it** with your private key (using ECDSA).
- The signature (`r`, `s`, `v`) is **included** in the transaction.
- That transaction is **broadcast to the entire Ethereum network**.
- It's then **stored publicly** on the blockchain.

**So any attacker can read `r` and `s`** for any transaction you ever signed.

Attackers can:

- Scrape all transactions from a given wallet.
- Compare `r` values across transactions.
- If `r` is reused → compute private key (even with just 2 transactions).
