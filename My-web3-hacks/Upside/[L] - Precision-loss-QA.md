## Lack of Input Validation in swap and processSwapFee Functions
This was because I found 10^6 and 10^18 numbers where divided/multiplied all together, this issue has been sucessfully exploited by other whitehat and was classified as Medium.

## Description
While reviewing processSwapFee I found out that neither function explicitly validates the input amounts or their decimal places. The contract relies on raw numbers (using parseUnits with hardcoded decimals) without checking if the provided amounts are valid or if the decimals match the token's expected decimals.

## Impact
~ I tested to fuzz the amount parameter of swap() to find edge cases but it led to nothing, the goal wasto buy low and sell high taking advantage of the lack of decimal validation but I eventually found out that for all the tests I was not able to extract more that the amount I bought.

While the contract does not allow price manipulation through incorrect decimals (it returns 0 USDC for such attempts), the lack of validation could cause confusion or errors for users.

## PoC
```typescript
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { ethers as hhethers } from "hardhat";
import { IERC20Metadata, UpsideMetaCoin, UpsideProtocol, UpsideStakingStub, USDCMock } from "../types";
import { expect } from "chai";

describe("ProcessSwapFee Tests", function () {
    let signers: HardhatEthersSigner[];
    let owner: HardhatEthersSigner;
    let user1: HardhatEthersSigner;
    let user2: HardhatEthersSigner;

    let stakingContract: UpsideStakingStub;
    let upsideProtocol: UpsideProtocol;
    let liquidityToken: USDCMock;
    let metaCoin: UpsideMetaCoin;

    const INITIAL_LIQUIDITY = hhethers.parseUnits("10000", 6); // 10,000 USDC
    const META_COIN_SUPPLY = hhethers.parseUnits("1000000", 18); // 1,000,000 MetaCoin

    before(async function () {
        signers = await hhethers.getSigners();
        owner = signers[0];
        user1 = signers[1];
        user2 = signers[2];

        // Deploy staking contract
        const stakingFactory = await hhethers.getContractFactory("UpsideStakingStub");
        stakingContract = await stakingFactory.deploy(owner.address);
        await stakingContract.setFeeDestinationAddress(owner.address);

        // Deploy protocol
        const protocolFactory = await hhethers.getContractFactory("UpsideProtocol");
        upsideProtocol = await protocolFactory.deploy(owner.address);

        // Deploy mock USDC
        const usdcFactory = await hhethers.getContractFactory("USDCMock");
        liquidityToken = await usdcFactory.deploy();

        // Initialize protocol
        await upsideProtocol.init(await liquidityToken.getAddress());
        await upsideProtocol.setStakingContractAddress(await stakingContract.getAddress());

        // Set fee parameters
        await upsideProtocol.setFeeInfo({
            tokenizeFeeDestinationAddress: owner.address,
            swapFeeDecayInterval: 86400, // 1 day
            tokenizeFeeEnabled: false,
            swapFeeStartingBp: 1000, // 10%
            swapFeeDecayBp: 100, // 1% decay per interval
            swapFeeFinalBp: 100, // 1% final fee
            swapFeeSellBp: 500, // 5% sell fee
            swapFeeDeployerBp: 2000 // 20% of fee goes to deployer
        });

        // Tokenize a URL
        await upsideProtocol.tokenize("https://test.com", await liquidityToken.getAddress());
        const metaCoinAddress = await upsideProtocol.urlToMetaCoinMap("https://test.com");
        metaCoin = await hhethers.getContractAt("UpsideMetaCoin", metaCoinAddress);

        // Mint USDC to users for testing
        await liquidityToken.mint(user1.address, hhethers.parseUnits("1000", 6));
        await liquidityToken.mint(user2.address, hhethers.parseUnits("1000", 6));
    });

    describe("Buy Operations", function () {
        it("should process fees correctly for a buy operation with 100 USDC", async function () {
            const buyAmount = hhethers.parseUnits("100", 6);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), buyAmount);

            const tx = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                true, // isBuy
                buyAmount,
                0, // minimumOut
                user1.address
            );

            const receipt = await tx.wait();
            const event = receipt?.logs.find(
                log => log.fragment && log.fragment.name === "SwapFeeProcessed"
            );

            expect(event).to.not.be.undefined;
            if (event) {
                const [metaCoinAddress, isBuy, secondsPassed, swapFeeBp, totalFee, feeToProtocol, feeToDeployer, feeToStakers] = event.args;
                
                // Verify fee calculations
                const expectedFee = (buyAmount * BigInt(1000)) / BigInt(10000); // 10% fee
                expect(totalFee).to.equal(expectedFee);
                expect(feeToProtocol).to.equal(expectedFee);
                expect(feeToDeployer).to.equal(0);
                expect(feeToStakers).to.equal(0);
            }
        });

        it("should process fees correctly for a buy operation with 1 USDC", async function () {
            const buyAmount = hhethers.parseUnits("1", 6);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), buyAmount);

            const tx = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                true,
                buyAmount,
                0,
                user1.address
            );

            const receipt = await tx.wait();
            const event = receipt?.logs.find(
                log => log.fragment && log.fragment.name === "SwapFeeProcessed"
            );

            expect(event).to.not.be.undefined;
            if (event) {
                const [metaCoinAddress, isBuy, secondsPassed, swapFeeBp, totalFee, feeToProtocol, feeToDeployer, feeToStakers] = event.args;
                
                // Verify fee calculations for small amount
                const expectedFee = (buyAmount * BigInt(1000)) / BigInt(10000);
                expect(totalFee).to.equal(expectedFee);
            }
        });
    });

    describe("Sell Operations", function () {
        it("should process fees correctly for a sell operation with 100 MetaCoin", async function () {
            // First buy some MetaCoin
            const buyAmount = hhethers.parseUnits("100", 6);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), buyAmount);
            await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                true,
                buyAmount,
                0,
                user1.address
            );

            // Now sell
            const sellAmount = hhethers.parseUnits("100", 18);
            await metaCoin.connect(user1).approve(await upsideProtocol.getAddress(), sellAmount);

            const tx = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                false, // isSell
                sellAmount,
                0,
                user1.address
            );

            const receipt = await tx.wait();
            const event = receipt?.logs.find(
                log => log.fragment && log.fragment.name === "SwapFeeProcessed"
            );

            expect(event).to.not.be.undefined;
            if (event) {
                const [metaCoinAddress, isBuy, secondsPassed, swapFeeBp, totalFee, feeToProtocol, feeToDeployer, feeToStakers] = event.args;
                
                // Verify fee calculations
                const expectedFee = (sellAmount * BigInt(500)) / BigInt(10000); // 5% sell fee
                const expectedDeployerFee = (expectedFee * BigInt(2000)) / BigInt(10000); // 20% to deployer
                const expectedStakerFee = expectedFee - expectedDeployerFee;

                expect(totalFee).to.equal(expectedFee);
                expect(feeToDeployer).to.equal(expectedDeployerFee);
                expect(feeToStakers).to.equal(expectedStakerFee);
            }
        });

        it("should process fees correctly for a sell operation with 1 MetaCoin", async function () {
            const sellAmount = hhethers.parseUnits("1", 18);
            await metaCoin.connect(user1).approve(await upsideProtocol.getAddress(), sellAmount);

            const tx = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                false,
                sellAmount,
                0,
                user1.address
            );

            const receipt = await tx.wait();
            const event = receipt?.logs.find(
                log => log.fragment && log.fragment.name === "SwapFeeProcessed"
            );

            expect(event).to.not.be.undefined;
            if (event) {
                const [metaCoinAddress, isBuy, secondsPassed, swapFeeBp, totalFee, feeToProtocol, feeToDeployer, feeToStakers] = event.args;
                
                // Verify fee calculations for small amount
                const expectedFee = (sellAmount * BigInt(500)) / BigInt(10000);
                const expectedDeployerFee = (expectedFee * BigInt(2000)) / BigInt(10000);
                const expectedStakerFee = expectedFee - expectedDeployerFee;

                expect(totalFee).to.equal(expectedFee);
                expect(feeToDeployer).to.equal(expectedDeployerFee);
                expect(feeToStakers).to.equal(expectedStakerFee);
            }
        });
    });

    describe("Edge Cases", function () {
        it("should handle zero amount swaps", async function () {
            const zeroAmount = 0;
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), zeroAmount);

            await expect(
                upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    true,
                    zeroAmount,
                    0,
                    user1.address
                )
            ).to.be.revertedWith("ERC20: transfer amount exceeds balance");
        });

        it("should handle very large amounts", async function () {
            const largeAmount = hhethers.parseUnits("1000000", 6); // 1M USDC
            await liquidityToken.mint(user1.address, largeAmount);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), largeAmount);

            const tx = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                true,
                largeAmount,
                0,
                user1.address
            );

            const receipt = await tx.wait();
            const event = receipt?.logs.find(
                log => log.fragment && log.fragment.name === "SwapFeeProcessed"
            );

            expect(event).to.not.be.undefined;
            if (event) {
                const [metaCoinAddress, isBuy, secondsPassed, swapFeeBp, totalFee, feeToProtocol, feeToDeployer, feeToStakers] = event.args;
                
                // Verify fee calculations for large amount
                const expectedFee = (largeAmount * BigInt(1000)) / BigInt(10000);
                expect(totalFee).to.equal(expectedFee);
            }
        });

        it("should revert for negative amounts", async function () {
            const negativeAmount = -1n;
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), hhethers.parseUnits("1000", 6));

            await expect(
                upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    true,
                    negativeAmount,
                    0,
                    user1.address
                )
            ).to.be.reverted;
        });

        it("should handle extremely large amounts (max uint256)", async function () {
            const maxAmount = 2n ** 256n - 1n;
            await liquidityToken.mint(user1.address, maxAmount);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), maxAmount);

            await expect(
                upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    true,
                    maxAmount,
                    0,
                    user1.address
                )
            ).to.be.reverted; // Should revert due to overflow in calculations
        });

        it("should handle amounts with incorrect decimals", async function () {
            // Try to swap with wrong decimal places (18 decimals instead of 6 for USDC)
            const wrongDecimalsAmount = hhethers.parseUnits("100", 18);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), wrongDecimalsAmount);

            await expect(
                upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    true,
                    wrongDecimalsAmount,
                    0,
                    user1.address
                )
            ).to.be.revertedWith("ERC20: transfer amount exceeds balance");
        });

        it("should handle very small amounts with different decimals", async function () {
            // Try to swap with very small amount (1 wei in USDC decimals)
            const smallAmount = 1n; // 1 wei in USDC decimals
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), smallAmount);

            await expect(
                upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    true,
                    smallAmount,
                    0,
                    user1.address
                )
            ).to.be.reverted; // Should revert due to precision loss
        });

        it("should handle decimal precision loss in fee calculations", async function () {
            // Try to swap with amount that would cause precision loss in fee calculation
            const precisionLossAmount = hhethers.parseUnits("0.000001", 6); // 1 USDC with 6 decimals
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), precisionLossAmount);

            await expect(
                upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    true,
                    precisionLossAmount,
                    0,
                    user1.address
                )
            ).to.be.reverted; // Should revert due to precision loss in fee calculation
        });

        it("should handle decimal mismatch in sell operations", async function () {
            // First buy some MetaCoin
            const buyAmount = hhethers.parseUnits("100", 6);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), buyAmount);
            await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                true,
                buyAmount,
                0,
                user1.address
            );

            // Now try to sell with wrong decimals
            const wrongDecimalsSellAmount = hhethers.parseUnits("100", 6); // Using USDC decimals for MetaCoin
            await metaCoin.connect(user1).approve(await upsideProtocol.getAddress(), wrongDecimalsSellAmount);

            await expect(
                upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    false,
                    wrongDecimalsSellAmount,
                    0,
                    user1.address
                )
            ).to.be.reverted; // Should revert due to decimal mismatch
        });

        it("should handle decimal overflow in bonding curve", async function () {
            // Try to swap with amount that would cause overflow in bonding curve
            const overflowAmount = hhethers.parseUnits("1000000000", 6); // 1B USDC
            await liquidityToken.mint(user1.address, overflowAmount);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), overflowAmount);

            await expect(
                upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    true,
                    overflowAmount,
                    0,
                    user1.address
                )
            ).to.be.reverted; // Should revert due to overflow in bonding curve
        });

        it("should handle amounts exceeding total supply", async function () {
            const exceedingAmount = hhethers.parseUnits("2000000", 6); // 2M USDC
            await liquidityToken.mint(user1.address, exceedingAmount);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), exceedingAmount);

            await expect(
                upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    true,
                    exceedingAmount,
                    0,
                    user1.address
                )
            ).to.be.revertedWith("InsufficientOutput");
        });

        it("should demonstrate price calculation issues with decimal precision", async function () {
            // First get the initial price by swapping 1 USDC
            const initialAmount = hhethers.parseUnits("1", 6);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), initialAmount);
            
            const initialSwap = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                true,
                initialAmount,
                0,
                user1.address
            );
            const initialReceipt = await initialSwap.wait();
            const initialEvent = initialReceipt?.logs.find(
                log => log.fragment && log.fragment.name === "Trade"
            );
            const initialAmountOut = initialEvent?.args[5]; // amountOut from Trade event
            
            // Now try to swap the same amount but with wrong decimals
            const wrongDecimalsAmount = hhethers.parseUnits("1", 18); // Using MetaCoin decimals for USDC
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), wrongDecimalsAmount);
            
            const secondSwap = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                true,
                wrongDecimalsAmount,
                0,
                user1.address
            );
            const secondReceipt = await secondSwap.wait();
            const secondEvent = secondReceipt?.logs.find(
                log => log.fragment && log.fragment.name === "Trade"
            );
            const secondAmountOut = secondEvent?.args[5]; // amountOut from Trade event
            
            // Compare the amounts - they should be different due to decimal mismatch
            console.log("Initial swap amount out:", hhethers.formatUnits(initialAmountOut, 18));
            console.log("Second swap amount out:", hhethers.formatUnits(secondAmountOut, 18));
            
            // The amounts should be significantly different due to decimal mismatch
            expect(initialAmountOut).to.not.equal(secondAmountOut);
        });

        it("should demonstrate price manipulation through decimal precision", async function () {
            // First get some MetaCoin
            const buyAmount = hhethers.parseUnits("100", 6);
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), buyAmount);
            await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                true,
                buyAmount,
                0,
                user1.address
            );

            // Now try to sell with different decimal precisions
            const sellAmount = hhethers.parseUnits("1", 18);
            await metaCoin.connect(user1).approve(await upsideProtocol.getAddress(), sellAmount);

            // First sell with correct decimals
            const firstSell = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                false,
                sellAmount,
                0,
                user1.address
            );
            const firstReceipt = await firstSell.wait();
            const firstEvent = firstReceipt?.logs.find(
                log => log.fragment && log.fragment.name === "Trade"
            );
            const firstAmountOut = firstEvent?.args[5]; // amountOut from Trade event

            // Now sell with wrong decimals (using USDC decimals)
            const wrongDecimalsSell = hhethers.parseUnits("1", 6);
            await metaCoin.connect(user1).approve(await upsideProtocol.getAddress(), wrongDecimalsSell);

            const secondSell = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                false,
                wrongDecimalsSell,
                0,
                user1.address
            );
            const secondReceipt = await secondSell.wait();
            const secondEvent = secondReceipt?.logs.find(
                log => log.fragment && log.fragment.name === "Trade"
            );
            const secondAmountOut = secondEvent?.args[5]; // amountOut from Trade event

            // Compare the amounts - they should be different due to decimal mismatch
            console.log("First sell amount out:", hhethers.formatUnits(firstAmountOut, 6));
            console.log("Second sell amount out:", hhethers.formatUnits(secondAmountOut, 6));
            
            // The amounts should be significantly different due to decimal mismatch
            expect(firstAmountOut).to.not.equal(secondAmountOut);
        });

        it("should demonstrate cumulative price impact from decimal precision", async function () {
            // Make multiple small swaps with different decimal precisions
            const smallAmount = hhethers.parseUnits("0.000001", 6); // Very small USDC amount
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), smallAmount * 10n);
            
            let totalAmountOut = 0n;
            
            // Make 10 swaps with correct decimals
            for(let i = 0; i < 10; i++) {
                const swap = await upsideProtocol.connect(user1).swap(
                    await metaCoin.getAddress(),
                    true,
                    smallAmount,
                    0,
                    user1.address
                );
                const receipt = await swap.wait();
                const event = receipt?.logs.find(
                    log => log.fragment && log.fragment.name === "Trade"
                );
                totalAmountOut += event?.args[5]; // amountOut from Trade event
            }
            
            console.log("Total amount out from 10 small swaps:", hhethers.formatUnits(totalAmountOut, 18));
            
            // Now make one swap with the total amount
            const totalAmount = smallAmount * 10n;
            await liquidityToken.connect(user1).approve(await upsideProtocol.getAddress(), totalAmount);
            
            const singleSwap = await upsideProtocol.connect(user1).swap(
                await metaCoin.getAddress(),
                true,
                totalAmount,
                0,
                user1.address
            );
            const singleReceipt = await singleSwap.wait();
            const singleEvent = singleReceipt?.logs.find(
                log => log.fragment && log.fragment.name === "Trade"
            );
            const singleAmountOut = singleEvent?.args[5]; // amountOut from Trade event
            
            console.log("Amount out from single large swap:", hhethers.formatUnits(singleAmountOut, 18));
            
            // The amounts should be different due to price impact
            expect(totalAmountOut).to.not.equal(singleAmountOut);
        });
    });
});
```

Logs:

```bash
npx hardhat test test/ProcessSwapFee.test.ts


  ProcessSwapFee Tests
    Buy Operations
      ✔ should process fees correctly for a buy operation with 100 USDC
      ✔ should process fees correctly for a buy operation with 1 USDC
    Sell Operations
      ✔ should process fees correctly for a sell operation with 100 MetaCoin
      ✔ should process fees correctly for a sell operation with 1 MetaCoin
    Edge Cases
      1) should handle zero amount swaps
      ✔ should handle very large amounts
      2) should revert for negative amounts
      3) should handle extremely large amounts (max uint256)
      4) should handle amounts with incorrect decimals
      5) should handle amounts exceeding total supply
```
While I wasn't able to exploit the decimal gap, we can still note that the function is not handling zero amount which should be considered in solidity SWE.
