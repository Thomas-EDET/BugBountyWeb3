The vulnerability involved a potential exploit that could allow an attacker to drain the base tokens from a pool by manipulating the token balance of a contract.

# Yield Protocol Logic Vulnerability Analysis

## Summary

The strategy contract’s `burn` function used on-chain LP token balances (`pool.balanceOf(address(this))`) to calculate redemptions, allowing an attacker to pre-fund the contract with extra LP tokens. By minting and immediately burning minimal shares, the attacker could extract far more LP tokens than they legitimately owned, systematically draining user funds.

## 1. Vulnerability Mechanism

### 1.1 Faulty Redemption Formula

After burning strategy shares, the contract computes LP tokens to return as:

```solidity
poolTokensObtained = 
    pool.balanceOf(address(this))  // on-chain LP token balance
  * burnt                         // number of shares just burned
  / totalSupply_;                 // total strategy shares before burn
```

Because `pool.balanceOf(address(this))` reflects **any** LP-token transfer, the attacker can inflate this value before redemption.

### 1.2 How the Attacker Inflates the Balance

An attacker directly transfers LP tokens into the strategy contract address prior to burning shares:

```solidity
poolToken.transfer(strategyAddress, 1);
```

This unauthorized deposit boosts the contract’s on-chain LP balance, which the flawed formula then uses to calculate redemptions. Because the formula blindly trusts `pool.balanceOf(address(this))`, any LP tokens sent to the contract—whether from users or attackers—are included in the payout computation. This gives the attacker a way to trick the contract into overpaying them during the burn.

### 1.3 Exploit Steps

 **Pre-fund** the strategy contract by transferring a small amount (e.g., 1 LP token) directly to the contract:

   ```solidity
   poolToken.transfer(strategyAddress, 1);
   ```

**Mint** a minimal number of strategy shares:
`strategy.mint(attackerAddress);`

**Immediately burn** those shares to trigger the flawed redemption logic:
strategy.burn(attackerAddress);

- The contract calculates the LP tokens to return using the inflated balance, giving the attacker more LP tokens than they should receive.
    
- **Repeat** the process—pre-funding and burning minimal shares—until the strategy contract is drained of its LP tokens and underlying base tokens.

## 2. Impact

- **Funds Exposed:** Approximately $950,000 in base tokens across Arbitrum and Ethereum pools.
    
- **Severity:** **Critical** — permits direct theft and draining of user funds.
    
- **Bounty:** The whitehat received $95,000 USDC (10% of at-risk funds) for responsible disclosure.
## 3. The Fix

The patch replaces the dynamic on-chain balance read with a **cached** snapshot variable (`poolCached_`) that updates only on legitimate deposits and withdrawals. The new redemption calculation uses:

`poolTokensObtained = poolCached_ * burnt / totalSupply_;`

This prevents any off-chain transfers from inflating redemption amounts.


##  My analysis
-> This is possible to drain all funds because `pool.balanceOf(address(this))` can be modified by the user by transfer asset to the contract modifing  the balance in the `poolTokensObtained = poolCached_ * burnt / totalSupply_;`  

After modifying the ratio  of  this equation, the tokens that can be burned are much higher than the initial deposit + the new transfer.  Therefore the attacker can withdraw multiple time making a net profit.

It's also  interesting how the PoC is deployed
attacker created a t.sol test than initiate a fork:
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

import "../../src/YieldProtocol/AttackContract.sol";

```solidity

pragma solidity ^0.8.13;
import "forge-std/Test.sol";

import "../../src/YieldProtocol/AttackContract.sol";


contract AttackTest is Test {
    AttackContract attackContract;

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/arbitrum", 92381336);

        attackContract = new AttackContract();
    }

    function testAttack() public {
        attackContract.initiateAttack();
    }
}
```

And  then using the contract forked to make the test.

References:https://github.com/immunefi-team/bugfix-reviews-pocs/blob/main/test/YieldProtocol/AttackTest.t.sol

https://medium.com/immunefi/yield-protocol-logic-error-bugfix-review-7b86741e6f50
