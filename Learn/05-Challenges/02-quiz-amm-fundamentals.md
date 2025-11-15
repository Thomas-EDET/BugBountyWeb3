Advanced Bug Bounty Quiz: AMMs, Trading, and Time-Ordering
Question 1:
A perpetual trading platform caches TWAP from a Uniswap v3 pool and uses it for liquidations. The TWAP is updated on-chain every 30 minutes.
What is a potential attack vector here?
A) Reentrancy
B) TWAP griefing via flash loan
C) Block timestamp manipulation
D) Frontrunning order execution

Question 2:
An AMM uses reserve0 and reserve1 values in getAmountOut() but does not sync them before each trade.
What vulnerability could this introduce?
A) Slippage miscalculation
B) Price manipulation via stale reserves
C) Gas griefing
D) Impermanent loss amplification

Question 3:
In a custom trading platform, a user submits a trade and pays a fee in a token whose value is volatile. The fee is calculated using block.timestamp to reference a price at time of submission.
What attack is possible here?
A) Timestamp spoofing
B) Fee arbitrage via MEV reordering
C) Reentrancy
D) Oracle sandwiching

Question 4:
A limit order book uses block.number to enforce a minimum delay before execution.
Which risk exists if the attacker controls the miner or uses a bundle?
A) They can skip the delay using tx.origin
B) They can frontrun and then backrun themselves
C) They can compress time by mining fast blocks
D) They can reorder and bypass the delay logic

Question 5:
A dual-token AMM uses token0 as a fee rebate token. The swap logic rebates the user after transferring tokens and before syncing the pool.
Which of these could be a valid exploit strategy?
A) Arbitraging the rebated amount on a DEX
B) Draining pool reserves via internal callback
C) Triggering reserve mismatch before syncing
D) Flash minting to bypass rebating logic



1b
2b
3b
4d
5c
