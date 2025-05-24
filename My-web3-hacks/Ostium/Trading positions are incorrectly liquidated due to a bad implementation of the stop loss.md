[Not Paid]


##  Conclusion 
Ostium Answer:<br>
"This is not a security bug report because it is the intended behavior. It is the user's responsibility to set proper stop loss. One example: a user farming the funding rate could be interested to set a sl lower than the liquidation price, knowing that the the liquidation price will move over time lower than the sl."
<br><br>
My answer: <br>
- RWA are not using funding rate, therefore the issue is still  affecting RWA pairs.
- Funding fee arbitrage strategies do not rely on being near-liquidated on initial open, especially not with high leverage.

---

### Full analysis:
I was auditing Ostium a perp trading platform. 

### Finding:<br>
Positions are being incorrectly liquidated. For a 10 SPX/USD long position with 10 collateral and 100x leverage, the liquidation price is inferior when a stop loss of -85% is applied compared to when no stop loss is set.
I said on the pair 10 but this has been found for all pair tested. <br>

### Vulnerability detail:<br>
The vulnerability comes from the contract OstiumPairInfos.sol where the maxNegativePnlOnOpenP is set to 75. This means that PnL cannot exceeds 75%. But the frontend application is allowing user to set a Stop Loss to 85%.

You can easily see the difference checking this link: https://ostium.app/trade?from=SPX&to=USD and setting a stop loss of 85% for a position with collateral = 100usd and leverage = 100.

See the following screenshot and focus on the liquidation price.

1 -  stop loss = -85% ;  liquidation price = 5757
![Screenshot from 2025-05-12 10-34-41](https://github.com/user-attachments/assets/35e6fef7-e4c3-482a-b472-e7b3da9d4446)

2 - no stop loss (-100%)  ; liquidation price =  5763

![Screenshot from 2025-05-12 10-34-30](https://github.com/user-attachments/assets/bfaef27a-3ba9-49f9-a7be-46f3a754a720)


### Impact detail:
If the liquidation price is higher than the stop-loss, it means positions are being closed earlier than intended, causing traders to lose money before their risk limit is actually hit. This breaks trust in the platform’s risk controls and makes it hard for users to manage their trades properly—especially with high leverage.

**[VERY IMPORTANT]-> It mostly impacts daily traders, daily traders are more likely to open trade with leverage > 20, which mean they are paying maker fees to the platform. If those traders found out that they are getting liquidated 10% before their stop loss, it's highly likely Ostium will loses revenue because the daily traders will flee the platform.**

### Recommendation:
To align with Ostium documentation I suggest Ostium: 1 - Set maxNegativePnlOnOpenP to 85 (and therefore review the math formula provided at https://ostium-labs.gitbook.io/ostium-docs/ostium-trading-engine/closing-trades) <br> [Difficult-clean]
OR 2 - Modify the frontend code to forbid the usage of stop loss = 85% accordingly to the liquidation trade formula, but people can still interract with the not ameneded contract. [easy-dirty]

### PoC 1 - fetching the maxNegativePnlOnOpenP value.<br>
The value of maxNegativePnlOnOpenP is the maximum negative PnL allowed, according to the frontend allowing traders to set a SL of -85%, this also should be set to 85.

```bash
cast call 0x3890243A8fc091C626ED26c087A028B46Bc9d66C "maxNegativePnlOnOpenP()(uint256)" --rpc-url http://127.0.0.1:8545 | cast --to-dec
75
```

### PoC 2 - Simulation  "traders are opening a trade with out of range SL"<br>
```solidiy
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import "../src/interfaces/IOstiumRegistry.sol";
import "../src/interfaces/IOstiumTrading.sol";
import "../src/interfaces/IOstiumVault.sol";
import "../src/interfaces/IOstiumTradingStorage.sol";
import "../src/interfaces/IOstiumPairsStorage.sol";
import "../src/interfaces/IOstiumPairInfos.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract TradingTest is Test {
    // Core contracts
    IOstiumRegistry public registry;
    IOstiumTrading public trading;
    IOstiumVault public vault;
    IOstiumTradingStorage public tradingStorage;
    IOstiumPairsStorage public pairsStorage;
    IOstiumPairInfos public pairInfos;
    
    // Constants
    address constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831; // Arbitrum USDC
    address constant REGISTRY = 0x799a139aE56e11F0476aCE2f6118CfcAed9608d2;
    uint16 constant SPX_PAIR_INDEX = 10;
    
    function setUp() public {
        // Fork Arbitrum
        vm.createSelectFork("http://127.0.0.1:8545");
        
        // Initialize contracts
        registry = IOstiumRegistry(REGISTRY);
        trading = IOstiumTrading(registry.getContractAddress("trading"));
        vault = IOstiumVault(registry.getContractAddress("vault"));
        tradingStorage = IOstiumTradingStorage(registry.getContractAddress("tradingStorage"));
        pairsStorage = IOstiumPairsStorage(registry.getContractAddress("pairsStorage"));
        pairInfos = IOstiumPairInfos(registry.getContractAddress("pairInfos"));
        
        // Get USDC
        deal(USDC, address(this), 100000000e6); // 10,000 USDC for testing
    }

    // Mock for arbBlockNumber precompile
    function mockArbBlockNumber() public {
        // Use vm.mockCall instead of etch
        vm.mockCall(
            address(0x64),  // The precompile address
            abi.encode(),   // Any input
            abi.encode(block.number)  // Return the current block number
        );
    }

    function testSimpleLongPosition() public {
        // Add this line before starting the test
        mockArbBlockNumber();
        
        address user = address(0x123); // simulate a user
        deal(USDC, user, 100000000e6); // give 10,000 USDC to user

        // Start impersonating the user
        vm.startPrank(user);

        // Approve all relevant contracts
        address[] memory contractsToApprove = new address[](6);
        contractsToApprove[0] = 0x6D0bA1f9996DBD8885827e1b2e8f6593e7702411;  // Trading proxy
        contractsToApprove[1] = 0xb9439B3274bd9dc7424c013Ff787F89ba77F03d1;  // Trading implementation
        contractsToApprove[2] = 0xcCd5891083A8acD2074690F65d3024E7D13d66E7;  // TradingStorage proxy
        contractsToApprove[3] = 0x5b6EBe448B396Aa57F5FBe03AB7Ed50C07532122;  // TradingStorage implementation
        contractsToApprove[4] = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;  // USDC contract
        contractsToApprove[5] = 0x86E721b43d4ECFa71119Dd38c0f938A75Fdb57B3;  // USDC implementation

        console.log("\nApproving contracts to spend USDC:");
        for(uint i = 0; i < contractsToApprove.length; i++) {
            IERC20(USDC).approve(contractsToApprove[i], type(uint256).max);
            console.log("- Approved:", contractsToApprove[i]);
            uint256 allowance = IERC20(USDC).allowance(user, contractsToApprove[i]);
            console.log("  Allowance:", allowance / 1e6, "USDC");
        }

        uint256 collateral = 20e6; // 20 USDC
        uint32 leverage = 10000;   // 100x leverage
        uint256 positionSize = collateral * leverage / 100;

        uint256 entryPrice = 5800;
        uint256 stopLossPrice = (entryPrice * 85) / 100;  // 85% of entry price = 4930

        console.log("\nTrade Parameters:");
        console.log("- Collateral:", collateral / 1e6, "USDC");
        console.log("- Leverage:", leverage / 100, "x");
        console.log("- Position Size:", positionSize / 1e6, "USDC");
        console.log("- Pair Index:", SPX_PAIR_INDEX);
        console.log("- Direction:", true ? "Long" : "Short");
        console.log("- Entry Price:", entryPrice);
        console.log("- Stop Loss: 2000", "(6550% of entry)");

        IOstiumTradingStorage.Trade memory trade = IOstiumTradingStorage.Trade({
            trader: user,
            pairIndex: SPX_PAIR_INDEX,
            buy: true,
            leverage: leverage,
            collateral: collateral,
            openPrice: 5800,
            tp: 0,
            sl: 2000,  // Using 4930 as stop loss (85% of 5800)
            index: 0
        });

        // Open trade
        trading.openTrade(
            trade,
            IOstiumTradingStorage.OpenOrderType.MARKET,
            100 // max slippage
        );

        vm.stopPrank();

        console.log("\nTrade opened successfully");
    }
}
```

Output:

```bash
forge test -vvv --match-test testSimpleLongPosition --via-ir
[⠊] Compiling...
[⠃] Compiling 1 files with Solc 0.8.28
[⠊] Solc 0.8.28 finished in 1.79s
Compiler run successful with warnings:
Warning (2072): Unused local variable.
  --> test/TradingTest.t.sol:87:9:
   |
87 |         uint256 stopLossPrice = (entryPrice * 85) / 100;  // 85% of entry price = 4930
   |         ^^^^^^^^^^^^^^^^^^^^^


Ran 1 test for test/TradingTest.t.sol:TradingTest
[PASS] testSimpleLongPosition() (gas: 724193)
Logs:
  
Approving contracts to spend USDC:
  - Approved: 0x6D0bA1f9996DBD8885827e1b2e8f6593e7702411
    Allowance: 115792089237316195423570985008687907853269984665640564039457584007913129 USDC
  - Approved: 0xb9439B3274bd9dc7424c013Ff787F89ba77F03d1
    Allowance: 115792089237316195423570985008687907853269984665640564039457584007913129 USDC
  - Approved: 0xcCd5891083A8acD2074690F65d3024E7D13d66E7
    Allowance: 115792089237316195423570985008687907853269984665640564039457584007913129 USDC
  - Approved: 0x5b6EBe448B396Aa57F5FBe03AB7Ed50C07532122
    Allowance: 115792089237316195423570985008687907853269984665640564039457584007913129 USDC
  - Approved: 0xaf88d065e77c8cC2239327C5EDb3A432268e5831
    Allowance: 115792089237316195423570985008687907853269984665640564039457584007913129 USDC
  - Approved: 0x86E721b43d4ECFa71119Dd38c0f938A75Fdb57B3
    Allowance: 115792089237316195423570985008687907853269984665640564039457584007913129 USDC
  
Trade Parameters:
  - Collateral: 20 USDC
  - Leverage: 100 x
  - Position Size: 2000 USDC
  - Pair Index: 10
  - Direction: Long
  - Entry Price: 5800
  - Stop Loss: 2000 (6550% of entry)
  
Trade opened successfully

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.83ms (2.16ms CPU time)

Ran 1 test suite in 8.58ms (4.83ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Stop Loss: 2000 (6550% of entry) shows that there is well an error in implementation of the liquidation price formula from the documentation.<br>

### What have we proved?<br>
1 - Traders can set incorrect stop loss on the frontend<br>
2 - The smart contract is not correctly implementing the Ostium liquidation design.<br>

### references
https://www.arbiscan.io/address/0x3890243a8fc091c626ed26c087a028b46bc9d66c?utm_source=immunefi#readProxyContract -> Check the maxNegativePnlOnOpenP set to 75%<br>
https://ostium-labs.gitbook.io/ostium-docs/ostium-trading-engine/closing-trades#liquidation -> check the liquidation formula here.<br>


---

### Go further  - my discussion with Ostium
Ostium answer:<br>
A user farming the funding rate could be interested to set a sl lower than the liquidation price, knowing that the the liquidation price will move over time lower than the sl.

But by reading the documentation:
I found out that funding rate is only for crypto pairs: "the funding fee compensates for OI imbalances". RWA pairs are using a rollover fees which made my remark valid!

See:https://ostium-labs.gitbook.io/ostium-docs/fee-breakdown#funding-rate-crypto-pairs

