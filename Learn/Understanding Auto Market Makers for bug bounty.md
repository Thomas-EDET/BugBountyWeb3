# Understanding Auto Market Makers for bug bounty

In this post we implement a simple AMM using `x*y=k`, and observe the outcome and potential vulnerabilities for bug bounties and research purposes.

---

**Automated Market Makers (AMMs)** use mathematical formulas that let users swap one token for another instantly, without needing a counterparty to be online.

### Uniswap AMM Formula

Uniswap v2 is a widely adopted AMM implementation that uses the constant product formula `x * y = k` ; it has plenty insane advantages for using it:

- Easy to understand and implement.
- **The curve never breaks** — there’s _always_ a price for any trade.
- It can be used as foundation for various DeFi projects.

But it has some specificities and drawbacks, if incorrectly understood, they can lead to **critical vulnerabilities**.

---

### `x*y=k` analysis in practice

To understand better this formula and its advantages and drawbacks, let's implement it in solidity as a smart contract.
#### Creation of the reserves x and y.
x and y are reserves of tokens, it's what we can call our *liquidity pool*.

```solidity
contract SimpleAMM {

uint256 public reserveX;
uint256 public reserveY;
```

For the moment the reserves are not linked together as suggest xy=k formula. This will be implemented later on.
#### Filling the reserve and defining K
K value is defined when the reserves are filled and when the reserves are linked to each others, here at the deployment of the contract:

```solidity
constructor() payable {
require(msg.value == 2 ether, "Must send exactly 2 ETH for initial liquidity");
reserveX = 1 ether;
reserveY = 1 ether;

emit LiquidityAdded(msg.sender, "X", 1 ether);
emit LiquidityAdded(msg.sender, "Y", 1 ether);
}
```
For this contract, the constructor requires the deployer to fill each reserve with 1 ether. In practice it means K = 1 ether * 1 ether; K = 10^18 * 10^18 = 10^36 wei.

Let's check that in practice:

```bash
=== Testing Constant Product Formula (x * y = k) ===
  Initial Reserve X: 1000000000000000000
  Initial Reserve Y: 1000000000000000000
  Initial K (x * y): 1000000000000000000000000000000000000
```

Good, we have defined a value for K. Let's link the reserves to stick to our formula xy=k
#### Creating an AMM in less than 3 minutes
From there creativity is key.<br>
1 - We can create a function to swap reserve X token to reserve Y token

```solidity
// NOT FOR PROD

function swapXforY() external payable {
require(msg.value > 0, "Must send ETH to swap");
require(reserveY > 0, "Insufficient Y reserve");

// Use the standard AMM formula: amountOut = (reserveY * amountIn) / (reserveX + amountIn)
// This maintains the constant product better than the division approach

uint256 amountOut = (reserveY * msg.value) / (reserveX + msg.value);
require(amountOut > 0, "No output amount");
require(amountOut < reserveY, "Insufficient Y reserve for swap");

// Update reserves
reserveX = reserveX + msg.value;
reserveY = reserveY - amountOut;

// Send ETH to user
payable(msg.sender).transfer(amountOut);
}
```


Here exactly we created a link between the reserves, when a user provide something into the reserve X the balance is performed by sending token from reserve Y to the user - maintaining the constant K.

2 - Perhaps we want to add more liquidity to the pool, or even better, allow others to add more liquidity to the pool like in Uniswap.

```solidity
function addLiquidityToX() external payable {
require(msg.value > 0, "Must send ETH to add liquidity");
reserveX += msg.value;
emit LiquidityAdded(msg.sender, "X", msg.value);
}

function addLiquidityToY() external payable {
require(msg.value > 0, "Must send ETH to add liquidity");
reserveY += msg.value;
emit LiquidityAdded(msg.sender, "Y", msg.value);
}
```

Here the implementation is flawed because to maintain K, the users should provide token to both reserve otherwise the reserve would be unbalanced and K would drift.

By creating those two features, bravo, you created a liquidity pool and people can swap two currencies, some other people can add liquidity to the pool, **in other terms you created an AMM!** 

In this implementation the two reserves contains ether so it's not interesting to swap, in real life it could be different tokens like eth/usdc:<br>
<img width="243" height="91" alt="Screenshot from 2025-08-06 15-00-07" src="https://github.com/user-attachments/assets/19cdd21b-a4af-4e0b-a502-e43515d44f07" />


**And that's it, in less than 3 minutes, you have now a usable AMM.** You know understand how powerful this formula `xy=k` is and how easy it is to implement.

---

But… this elegant formula isn’t without its challenges. Let's explain the major one : The slippagee.
#### Slippage explanation 
Let's use our swap function and simulate a user providing 0.5 eth to reserve X. To maintain K, some token from reserveY need to be sent to the user.

Large Swap Test:
```bash
 forge test -vv --match-test testSwapXforY
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/SimpleAMM.t.sol:SimpleAMMTest
[PASS] testSwapXforY() (gas: 57202)
Logs:
  
=== Initial State ===
  User ETH balance: 1000000000000000000
  Reserve X: 1000000000000000000
  Reserve Y: 1000000000000000000
  
Swapping 0.5 ETH from X to Y
  Expected output from Y reserve: 333333333333333333
  
=== Final State ===
  User ETH balance: 833333333333333333
  Reserve X: 1500000000000000000
  Reserve Y: 666666666666666667
  
Constant product (k) after swap: 1000000000000000000500000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 974.12µs (426.49µs CPU time)
```

We notice:<br>
1 - The user sent 0.5 ETH and only got back ~0.333... ETH — that’s a huge 33% loss!<br>
2 - k slightly increase while our formula is factually correct.<br>

Small Swap Test:
```bash
=== Initial State (Small Swap) ===
  User ETH balance: 1000000000000000000
  Reserve X: 1000000000000000000
  Reserve Y: 1000000000000000000
  
Swapping 0.0001 ETH from X to Y
  Expected output from Y reserve: 99990000999900
  
=== Final State (Small Swap) ===
  User ETH balance: 999999990000999900
  Reserve X: 1000100000000000000
  Reserve Y: 999900009999000100
  
Constant product (k) after small swap: 1000000000000000000010000000000000000
```

Comparison:

| Metric                    | Large Swap (0.5 ETH) | Small Swap (0.0001 ETH) |
| ------------------------- | -------------------- | ----------------------- |
| Input Amount              | 0.5 ETH              | 0.0001 ETH              |
| Output Received           | 0.333333 ETH         | 0.00009999 ETH          |
| Effective Price (X per Y) | **1.5**              | **1.0001**              |
| Slippage                  | 50%                  | 0.01%                   |
| Value Lost (%)            | 33.3%                | 0.01%                   |
| K Drift                   | +0.00005%            | +0.00000001%            |
|                           |                      |                         |
#### Main observations
📉 Discussing the Economic loss:
That loss from the big swap isn't **"stolen"** — it’s a **core feature** of constant product AMMs (like Uniswap v2) due to **slippage**, and it **becomes more punishing when trading large amounts relative to the pool size**. The root cause of this lost is because of the curve shape becomes extremely steep near the edges.

Let's visualize  the curve:
<img width="447" height="321" alt="Screenshot from 2025-08-08 11-46-42" src="https://github.com/user-attachments/assets/065ef634-3e49-4d02-a1ac-dcf98ec45d1e" />


Based on the curve xy=k, we understand that, NEVER, we will receive the same amount of token provided. This can be easily visualized because the segment orange and the segment in blue are not the same length.

🧮 Discussing Why K Doesn't Stay Constant?
Even though the math is sound, **`K` increases slightly**. Why?

- Solidity uses **integer division** → always rounds down
- When computing `amountOut`, some precision is lost
- That small loss is retained in the reserves → K slightly increases

---

We understand better the formula and its main specificity. Where to focus in bug bounty?

### Bug bounty tips

#### 1 - Detecting K drift.

**Keeping `k` constant preserves trust, math, and fairness in the system.**  
It enforces a balance between token reserves and protects both users and LPs.

if K decrease: It would suggest value is leaking from the system — potentially exploitable.
if K increase: It means worse rates for users swapping, pool will not be used anymore as not competitive anymore.

In the slippage description the K drift has been calculated as follow:

```solidity
uint256 kBefore = reserve0 * reserve1;
// execute swap
uint256 kAfter = newReserve0 * newReserve1;

```

**Bug bounty example:**  
A pool updates reserves _after_ sending tokens to the user. A malicious token calls back, exploiting a reentrancy and manipulates reserves before finalizing → `k` breaks.

#### 2 - MEV Scenarios – Miner Extractable Value
Even if `k` is maintained, traders can lose big through frontrunning or sandwiching. Some protocols wrongly assume MEV is just “market risk” and not exploitable — but if you can _guarantee profit_ via predictable swaps, it’s a valid finding.

**Hunting checklist:**

- **Front-run opportunity:**
    - Large swaps with predictable calldata.
    - Missing anti-MEV measures (e.g., `deadline` checks, `permit` usage with delays).
- **Sandwichable swaps:**
    - No slippage protection → you can front-run, push price up, let victim swap at bad rate, back-run for profit.
- **Gas golfing:**
    - Identify swap paths cheaper for attacker to sandwich due to low gas, making it more profitable.        
- **Flashbots blind spot:**
    - Check if they use private transaction relays; if not, swaps are public and snipeable.

**Bug bounty example:**  
A stable pool with no `minAmountOut` check lets attacker sandwich swaps infinitely until LPs bear the impermanent loss.


#### 3  - **Special Watchlist for `xy = k` Pools**

When reviewing an AMM contract:

- **Fee-on-transfer tokens** — Mess with reserve updates, cause `k` drift.
- **Tokens with `decimals` ≠ 18** — Precision loss may create exploitable gaps.
- **Incorrect `minLiquidity` logic** — First LP can mint an unfairly large share.
- **No lock on LP minting** — Front-run pool creation to own majority liquidity.
- **Paired with illiquid token** — Easier to manipulate price due to low depth.
- **AMM price as Oracle** —  (`reserve1 / reserve0`) without safeguards.


xy=k is a powerful formula but other formula are used in popular AMMs.

---
### Other bonding curves and associated formula in real world

#### UniswapV2
**Formula**: `x * y = k`

- Price impact increases **non-linearly** with trade size.
- Encourages early liquidity, penalizes large trades.
- Infinite liquidity at extreme prices → arbitrageurs enforce price discovery.

<img width="647" height="510" alt="Screenshot from 2025-08-06 09-57-02" src="https://github.com/user-attachments/assets/d2f26692-f7ff-42ef-a56a-617a99efdfe7" />



#### Curve finance
**Formula**: `A * n * Σx_i + D = A * D * n + D³ / (n^n * Πx_i)

- **Near-peg** assets (e.g., USDC/USDT/DAI): minimal slippage.
- Outside peg → behaves more like `x*y=k` to ensure reserves aren’t drained.
- Parameter `A` (amplification coefficient) controls how tightly the pool sticks to the peg.
- **Downside**: fails if one asset in the pool depegs badly (e.g., UST crash).

**Use Case**: Stablecoins, Liquid Staking Tokens (LSTs like stETH)

<img width="349" height="216" alt="Screenshot from 2025-08-06 11-21-11" src="https://github.com/user-attachments/assets/51f9af2a-3062-4e21-8618-72b9bb037a3b" />


Problem with x + y = k is depeg. If one token depegs, users can **drain all of the good asset** from the pool without any resistance. That's why we can clearly see that the curve from curve.finance is acting more like x y = k.

I was expecting to see more volume in Curve finance pool than in uniswap but uniswap is now using concentrated pools and active management resulting in higher earning fees for the LProviders.

Finance Tips: To swap I'll recommend using curve.finance; to provide I'll recommend uniswap despite their risks is slightly more important; because of concentrated liquidity mechanisms.
#### Other important curves 
Balancer — Weighted Constant Product
**Formula**:`k = Π balance_i ^ weight_i` -> You can do 80/20, 60/40 pools

BancorV2 - bonding curve + oracle
Adjusts **weights** and **reserves dynamically** using **oracles** and **impermanent loss protection** mechanisms. 
Formula: xy=k
#### Shell Protocol - Copper - Centrifuge - Primitive finance
Those projects are using custom AMMs bonding curves - I recommend following those projects.

---

Find the resources I used and my code for the demo bellow. Thanks following and supporting me so far :heart:.

---

Contracts and tests used for the demo

Contact AMMs
```soldidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleAMM {
    uint256 public reserveX;
    uint256 public reserveY;

    event LiquidityAdded(address indexed provider, string reserve, uint256 amount);
    event Swap(address indexed user, string fromReserve, string toReserve, uint256 amountIn, uint256 amountOut);

    constructor() payable {
        require(msg.value == 4 ether, "Must send exactly 4 ETH for initial liquidity");
        reserveX = 2 ether;
        reserveY = 2 ether;
        emit LiquidityAdded(msg.sender, "X", 2 ether);
        emit LiquidityAdded(msg.sender, "Y", 2 ether);
    }

    function addLiquidityToX() external payable {
        require(msg.value > 0, "Must send ETH to add liquidity");
        reserveX += msg.value;
        emit LiquidityAdded(msg.sender, "X", msg.value);
    }

    function addLiquidityToY() external payable {
        require(msg.value > 0, "Must send ETH to add liquidity");
        reserveY += msg.value;
        emit LiquidityAdded(msg.sender, "Y", msg.value);
    }

    // Get the current constant product (k = x * y)
    function getK() public view returns (uint256) {
        return reserveX * reserveY;
    }

    // Swap from X to Y: Send ETH to X reserve, get ETH from Y reserve
    function swapXforY() external payable {
        require(msg.value > 0, "Must send ETH to swap");
        require(reserveY > 0, "Insufficient Y reserve");
        
        // Use the standard AMM formula: amountOut = (reserveY * amountIn) / (reserveX + amountIn)
        // This maintains the constant product better than the division approach
        uint256 amountOut = (reserveY * msg.value) / (reserveX + msg.value);
        //uint256 amountOut = reserveY - (reserveX * reserveY) / (reserveX + msg.value);
        require(amountOut > 0, "No output amount");
        require(amountOut < reserveY, "Insufficient Y reserve for swap");
        
        // Update reserves
        reserveX = reserveX + msg.value;
        reserveY = reserveY - amountOut;
        
        // Send ETH to user
        payable(msg.sender).transfer(amountOut);
        
        emit Swap(msg.sender, "X", "Y", msg.value, amountOut);
    }

    // Swap from Y to X: Send ETH to Y reserve, get ETH from X reserve  
    function swapYforX() external payable {
        require(msg.value > 0, "Must send ETH to swap");
        require(reserveX > 0, "Insufficient X reserve");
        
        // Use the standard AMM formula: amountOut = (reserveX * amountIn) / (reserveY + amountIn)
        uint256 amountOut = (reserveX * msg.value) / (reserveY + msg.value);
        require(amountOut > 0, "No output amount");
        require(amountOut < reserveX, "Insufficient X reserve for swap");
        
        // Update reserves
        reserveY = reserveY + msg.value;
        reserveX = reserveX - amountOut;
        
        // Send ETH to user
        payable(msg.sender).transfer(amountOut);
        
        emit Swap(msg.sender, "Y", "X", msg.value, amountOut);
    }

    // Helper function to calculate output amount for a given input (without executing)
    function getAmountOut(uint256 amountIn, uint256 reserveIn, uint256 reserveOut) 
        public pure returns (uint256) {
        require(amountIn > 0, "Input amount must be positive");
        require(reserveIn > 0 && reserveOut > 0, "Reserves must be positive");
        
        // Use the same formula as swap functions: amountOut = (reserveOut * amountIn) / (reserveIn + amountIn)
        return (reserveOut * amountIn) / (reserveIn + amountIn);
    }
}
```

Tests
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/SimpleAMM.sol";

contract SimpleAMMTest is Test {
    SimpleAMM public amm;
    
    function setUp() public {
        // Give the test contract some ETH to deploy the AMM
        vm.deal(address(this), 4 ether);
        
        // Deploy AMM with 4 ETH (2 ETH for each reserve)
        amm = new SimpleAMM{value: 4 ether}();
    }
    
    function testInitialReserves() public {
        // Display initial reserves
        console.log("Initial Reserve X:", amm.reserveX());
        console.log("Initial Reserve Y:", amm.reserveY());
        
        // Verify both reserves are exactly 2 ETH
        assertEq(amm.reserveX(), 2 ether, "Reserve X should be 2 ETH");
        assertEq(amm.reserveY(), 2 ether, "Reserve Y should be 2 ETH");
        
        // Verify total ETH in contract
        assertEq(address(amm).balance, 4 ether, "Contract should hold 4 ETH total");
        
        console.log("Test passed: Both reserves initialized with 2 ETH each");
    }
    
    function testSwapXforY() public {
        // Create a user with 1 ETH
        address user = makeAddr("user");
        vm.deal(user, 1 ether);
        
        // Display initial state
        console.log("\n=== Initial State ===");
        console.log("User ETH balance:", user.balance);
        console.log("Reserve X:", amm.reserveX());
        console.log("Reserve Y:", amm.reserveY());
        
        // Calculate expected output for 0.5 ETH swap
        uint256 swapAmount = 0.5 ether;
        uint256 expectedOutput = amm.getAmountOut(swapAmount, amm.reserveX(), amm.reserveY());
        uint256 initialK = amm.getK();
        console.log("\nSwapping 0.5 ETH from X to Y");
        console.log("Expected output from Y reserve:", expectedOutput);
        
        // Perform the swap
        vm.prank(user);
        amm.swapXforY{value: swapAmount}();
        
        // Display final state
        console.log("\n=== Final State ===");
        console.log("User ETH balance:", user.balance);
        console.log("Reserve X:", amm.reserveX());
        console.log("Reserve Y:", amm.reserveY());
        
        // Verify the swap was successful
        assertEq(amm.reserveX(), 2 ether + swapAmount, "Reserve X should have increased by 0.5 ETH");
        assertEq(amm.reserveY(), 2 ether - expectedOutput, "Reserve Y should have decreased by the output amount");
        assertEq(user.balance, 1 ether - swapAmount + expectedOutput, "User should have 1 ETH minus swap plus output");
        
        // Verify constant product formula
        uint256 k = amm.getK();
        console.log("\nConstant product (k) after swap:", k);
        assertGe(k, initialK, "k should not decrease after swap");
    }

    function testSwapXforY_SmallAmount() public {
        // Create a user with 1 ETH
        address user = makeAddr("userSmall");
        vm.deal(user, 1 ether);

        // Display initial state
        console.log("\n=== Initial State (Small Swap) ===");
        console.log("User ETH balance:", user.balance);
        console.log("Reserve X:", amm.reserveX());
        console.log("Reserve Y:", amm.reserveY());

        // Calculate expected output for 0.0001 ETH swap
        uint256 swapAmount = 0.0001 ether;
        uint256 expectedOutput = amm.getAmountOut(swapAmount, amm.reserveX(), amm.reserveY());
        uint256 initialK = amm.getK();
        console.log("\nSwapping 0.0001 ETH from X to Y");
        console.log("Expected output from Y reserve:", expectedOutput);

        // Perform the swap
        vm.prank(user);
        amm.swapXforY{value: swapAmount}();

        // Display final state
        console.log("\n=== Final State (Small Swap) ===");
        console.log("User ETH balance:", user.balance);
        console.log("Reserve X:", amm.reserveX());
        console.log("Reserve Y:", amm.reserveY());

        // Verify the swap was successful
        assertEq(amm.reserveX(), 2 ether + swapAmount, "Reserve X should have increased by swap amount");
        assertEq(amm.reserveY(), 2 ether - expectedOutput, "Reserve Y should have decreased by the output amount");
        assertEq(user.balance, 1 ether - swapAmount + expectedOutput, "User should have initial ETH minus swap plus output");

        // Verify constant product formula
        uint256 k = amm.getK();
        console.log("\nConstant product (k) after small swap:", k);
        assertGe(k, initialK, "k should not decrease after swap");
    }

    function testManySwapsXforY() public {
        address user = makeAddr("userMany");
        vm.deal(user, 10 ether); // Give user enough ETH for many swaps
        uint256 swapAmount = 0.1 ether;
        uint256 numSwaps = 20;
        uint256[] memory kValues = new uint256[](numSwaps + 1);

        // Initial k
        kValues[0] = amm.getK();
        uint256 lastK = kValues[0];
        console.log("Initial k:", lastK);

        for (uint256 i = 1; i <= numSwaps; i++) {
            // Calculate expected output (not used for assertions, just for info)
            uint256 expectedOutput = amm.getAmountOut(swapAmount, amm.reserveX(), amm.reserveY());
            // Perform the swap
            vm.prank(user);
            amm.swapXforY{value: swapAmount}();
            // Get new k
            uint256 k = amm.getK();
            kValues[i] = k;
            console.log(string(abi.encodePacked("k after swap ", vm.toString(i), ": ")), k);
            // Assert k does not decrease
            assertGe(k, lastK, "k should not decrease after swap");
            lastK = k;
        }
        // Optionally, print all k values at the end
        console.log("\nAll k values after each swap:");
        for (uint256 i = 0; i <= numSwaps; i++) {
            console.log(string(abi.encodePacked("k[", vm.toString(i), "] = ")), kValues[i]);
        }
    }

    function testManyLargeSwapsXforY() public {
        address user = makeAddr("userLargeMany");
        vm.deal(user, 10 ether); // Give user enough ETH for many swaps
        uint256 swapAmount = 1.5 ether; // Large swap relative to pool
        uint256 numSwaps = 5;
        uint256[] memory kValues = new uint256[](numSwaps + 1);
        uint256[] memory userBalances = new uint256[](numSwaps + 1);
        uint256[] memory reserveXValues = new uint256[](numSwaps + 1);
        uint256[] memory reserveYValues = new uint256[](numSwaps + 1);

        // Initial k, user balance, and reserves
        kValues[0] = amm.getK();
        userBalances[0] = user.balance;
        reserveXValues[0] = amm.reserveX();
        reserveYValues[0] = amm.reserveY();
        uint256 lastK = kValues[0];
        console.log("Initial k:", lastK);
        console.log("Initial user balance:", userBalances[0]);
        console.log("Initial reserveX:", reserveXValues[0]);
        console.log("Initial reserveY:", reserveYValues[0]);

        for (uint256 i = 1; i <= numSwaps; i++) {
            // Calculate expected output (not used for assertions, just for info)
            uint256 expectedOutput = amm.getAmountOut(swapAmount, amm.reserveX(), amm.reserveY());
            // Perform the swap
            vm.prank(user);
            amm.swapXforY{value: swapAmount}();
            // Get new k, user balance, and reserves
            uint256 k = amm.getK();
            uint256 userBal = user.balance;
            uint256 reserveX = amm.reserveX();
            uint256 reserveY = amm.reserveY();
            kValues[i] = k;
            userBalances[i] = userBal;
            reserveXValues[i] = reserveX;
            reserveYValues[i] = reserveY;
            console.log(string(abi.encodePacked("k after large swap ", vm.toString(i), ": ")), k);
            console.log(string(abi.encodePacked("user balance after large swap ", vm.toString(i), ": ")), userBal);
            console.log(string(abi.encodePacked("reserveX after large swap ", vm.toString(i), ": ")), reserveX);
            console.log(string(abi.encodePacked("reserveY after large swap ", vm.toString(i), ": ")), reserveY);
            // Assert k does not decrease
            assertGe(k, lastK, "k should not decrease after swap");
            lastK = k;
        }
        // Optionally, print all k values, user balances, and reserves at the end
        console.log("\nAll k values after each large swap:");
        for (uint256 i = 0; i <= numSwaps; i++) {
            console.log(string(abi.encodePacked("k[", vm.toString(i), "] = ")), kValues[i]);
        }
        console.log("\nAll user balances after each large swap:");
        for (uint256 i = 0; i <= numSwaps; i++) {
            console.log(string(abi.encodePacked("userBalance[", vm.toString(i), "] = ")), userBalances[i]);
        }
        console.log("\nAll reserveX values after each large swap:");
        for (uint256 i = 0; i <= numSwaps; i++) {
            console.log(string(abi.encodePacked("reserveX[", vm.toString(i), "] = ")), reserveXValues[i]);
        }
        console.log("\nAll reserveY values after each large swap:");
        for (uint256 i = 0; i <= numSwaps; i++) {
            console.log(string(abi.encodePacked("reserveY[", vm.toString(i), "] = ")), reserveYValues[i]);
        }
    }
}

```

---
Resources:
https://www.curve.finance/files/stableswap-paper.pdf
https://www.dydx.xyz/crypto-learning/bonding-curve
https://speedrunethereum.com/guides/solidity-bonding-curves-token-pricing
https://docs.coinweb.io/learn/protocol/custom-tokens/token-bonding-curves

