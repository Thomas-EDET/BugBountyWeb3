## Finding description and impact
I found out that the setMetaCoinWhitelist function in UpsideProtocol.sol lacks validation for array length consistency between its three input parameters: _metaCoinAddresses, _walletAddresses, and _isWhitelisted. This oversight can lead to transaction reverts when the arrays have mismatched lengths.

The function iterates over _metaCoinAddresses.length but accesses elements from all three arrays without checking if their lengths match. Therefore I write a test to verify it can lead to a out-of-bounds array access, causing the transaction to revert.

Potential DoS: If the owner accidentally or maliciously calls this function with mismatched arrays, it could prevent legitimate whitelist updates until the issue is fixed.

## Severity rating rational:
The issue affects the whitelist update functionality, which is important for access control.
However, I believe the impact is limited to incomplete updates rather than a complete failure or exploitation of the proto'.

## Recommended mitigation steps
You can set up your function with clear a check to prevent out-of-bounds array access and also error message when array lengths don't match.
```solidity
function setMetaCoinWhitelist(
    address[] calldata _metaCoinAddresses,
    address[] calldata _walletAddresses,
    bool[] calldata _isWhitelisted
) external onlyOwner {
    require(
        _metaCoinAddresses.length == _walletAddresses.length && 
        _walletAddresses.length == _isWhitelisted.length,
        "Array lengths must match"
    );
    
    for (uint256 i; i < _metaCoinAddresses.length; ) {
        metaCoinWhitelistMap[_metaCoinAddresses[i]][_walletAddresses[i]] = _isWhitelisted[i];
        emit MetaCoinWhitelistSet(_metaCoinAddresses[i], _walletAddresses[i], _isWhitelisted[i]);
        unchecked {
            ++i;
        }
    }
}
```

## Proof of Concept
I've written a full javascript PoC following the guidelines from the readme.txt

The function is inconsistent: sometimes it reverts, sometimes it silently ignores extra data.
This can lead to confusion, incomplete updates, and potential security or operational issues.

```typescript
import { expect } from "chai";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { ethers as hhethers } from "hardhat";
import { IERC20Metadata, UpsideMetaCoin, UpsideProtocol, UpsideStakingStub } from "../types";

describe("Whitelist Array Length Tests", function () {
    let signers: HardhatEthersSigner[];
    let owner: HardhatEthersSigner;
    let user1: HardhatEthersSigner;
    let user2: HardhatEthersSigner;

    let stakingContract: UpsideStakingStub;
    let upsideProtocol: UpsideProtocol;
    let liquidityToken: IERC20Metadata;
    let testToken: UpsideMetaCoin;

    before(async function () {
        // Get signers
        signers = await hhethers.getSigners();
        owner = signers[0];
        user1 = signers[1];
        user2 = signers[2];

        // Deploy staking contract
        const stakingFactory = await hhethers.getContractFactory("UpsideStakingStub");
        stakingContract = await stakingFactory.deploy(owner.address);

        // Deploy protocol
        const protocolFactory = await hhethers.getContractFactory("UpsideProtocol");
        upsideProtocol = await protocolFactory.deploy(owner.address);

        // Deploy mock USDC
        const usdcFactory = await hhethers.getContractFactory("USDCMock");
        liquidityToken = await usdcFactory.deploy();

        // Initialize protocol
        await upsideProtocol.connect(owner).init(await liquidityToken.getAddress());
        await upsideProtocol.connect(owner).setStakingContractAddress(await stakingContract.getAddress());

        // Create a test token
        await upsideProtocol.connect(owner).tokenize("https://test.com", await liquidityToken.getAddress());
        testToken = await hhethers.getContractAt(
            "UpsideMetaCoin",
            await upsideProtocol.urlToMetaCoinMap("https://test.com")
        );
    });

    describe("Array Length Mismatch Tests", function () {
        it("should revert when metaCoinAddresses length is greater than walletAddresses length", async function () {
            await expect(
                upsideProtocol.connect(owner).setMetaCoinWhitelist(
                    [await testToken.getAddress(), await testToken.getAddress()], // 2 addresses
                    [user1.address], // 1 address
                    [true] // 1 boolean
                )
            ).to.be.reverted;
        });

        it("should revert when walletAddresses length is greater than isWhitelisted length", async function () {
            await expect(
                upsideProtocol.connect(owner).setMetaCoinWhitelist(
                    [await testToken.getAddress()], // 1 address
                    [user1.address, user2.address], // 2 addresses
                    [true] // 1 boolean
                )
            ).to.be.reverted;
        });

        it("should revert when isWhitelisted length is greater than metaCoinAddresses length", async function () {
            await expect(
                upsideProtocol.connect(owner).setMetaCoinWhitelist(
                    [await testToken.getAddress()], // 1 address
                    [user1.address], // 1 address
                    [true, false] // 2 booleans
                )
            ).to.be.reverted;
        });

        it("should succeed when all array lengths match", async function () {
            await expect(
                upsideProtocol.connect(owner).setMetaCoinWhitelist(
                    [await testToken.getAddress()],
                    [user1.address],
                    [true]
                )
            ).to.not.be.reverted;

            // Verify whitelist was set correctly
            const isWhitelisted = await upsideProtocol.metaCoinWhitelist(await testToken.getAddress(), user1.address);
            expect(isWhitelisted).to.be.true;
        });

        it("should handle multiple entries when all array lengths match", async function () {
            await expect(
                upsideProtocol.connect(owner).setMetaCoinWhitelist(
                    [await testToken.getAddress(), await testToken.getAddress()],
                    [user1.address, user2.address],
                    [true, false]
                )
            ).to.not.be.reverted;

            // Verify whitelist was set correctly for both users
            const isWhitelisted1 = await upsideProtocol.metaCoinWhitelist(await testToken.getAddress(), user1.address);
            const isWhitelisted2 = await upsideProtocol.metaCoinWhitelist(await testToken.getAddress(), user2.address);
            expect(isWhitelisted1).to.be.true;
            expect(isWhitelisted2).to.be.false;
        });

        it("should revert when walletAddresses length is greater than isWhitelisted length", async function () {
            await expect(
                upsideProtocol.connect(owner).setMetaCoinWhitelist(
                    [await testToken.getAddress()], // 1 address
                    [user1.address, user1.address], // 2 addresses
                    [true] // 1 boolean
                )
            ).to.be.reverted;
        });
    });
});
```
Here is the output:
```bash
npx hardhat test test/WhitelistArray.test.ts


  Whitelist Array Length Tests
    Array Length Mismatch Tests
      ✔ should revert when metaCoinAddresses length is greater than walletAddresses length
      1) should revert when walletAddresses length is greater than isWhitelisted length
      2) should revert when isWhitelisted length is greater than metaCoinAddresses length
      ✔ should succeed when all array lengths match
      ✔ should handle multiple entries when all array lengths match
      3) should revert when walletAddresses length is greater than isWhitelisted length


  3 passing (474ms)
  3 failing

  1) Whitelist Array Length Tests
       Array Length Mismatch Tests
         should revert when walletAddresses length is greater than isWhitelisted length:
     AssertionError: Expected transaction to be reverted
      at async Context.<anonymous> (test/WhitelistArray.test.ts:60:13)

  2) Whitelist Array Length Tests
       Array Length Mismatch Tests
         should revert when isWhitelisted length is greater than metaCoinAddresses length:
     AssertionError: Expected transaction to be reverted
      at async Context.<anonymous> (test/WhitelistArray.test.ts:70:13)

  3) Whitelist Array Length Tests
       Array Length Mismatch Tests
         should revert when walletAddresses length is greater than isWhitelisted length:
     AssertionError: Expected transaction to be reverted
      at async Context.<anonymous> (test/WhitelistArray.test.ts:110:13)
```

## Links to affected code
[UpsideProtocol.sol#L321-L333](https://github.com/code-423n4/2025-05-upside/blob/main/contracts/UpsideProtocol.sol#L321-L333)
