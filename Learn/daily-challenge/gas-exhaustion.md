### Challenge The Ethernaut Denial

This is a simple wallet that drips funds over time. You can withdraw the funds slowly by becoming a withdrawing partner.
If you can deny the owner from withdrawing funds when they call withdraw() (whilst the contract still has funds, and the transaction is of 1M gas or less) you will win this level.

### .call explanation
.call is a low-level function in Solidity.<br>
It allows you to: Send ETH, Call a function by raw bytes (manual ABI), Forward gas manually.<br>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Denial {
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint256 timeLastWithdrawn;
    mapping(address => uint256) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint256 amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value: amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] += amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```
My first observations:
- The four functions are public
- We can pass data in only one function : setWithdrawPartner(_partner)
- We have one level call `partner.call{value: amountToSend}("");`
- CEI is not followed allowing potential reentrancy BUT there is not transfer to partners nor balance update possible.

When I saw the .call function I knew it was our entry point.<br>
.call always call a fallback() ; in the fallback we can merely add a loop that burn gas

Example of a fallback function that burn gas
```solidity
fallback() external payable {
    for (uint256 i = 0; i < 1000; i++) {
        // do some expensive computation
        keccak256(abi.encodePacked(i));
    }
}
```
~Hashing is always expensive we can use keccak function

After deploying our denial contract we can call withdraw:
`await denialContract.withdraw({ gasLimit: 1000000 });`

The thing I learnt that helped me:
- Low level call does not revert on failure
- Understanding that the return value of .call is not checked will continue the execution

Thing I learnt after that:<br>
- ALWAYS set a gas limit for untrusted contract.
- a full function can revert but some low level calls still might succeed.
- As a dev, always check the return value after a low level call:<br>
```solidity
(bool success, ) = partner.call{value: amountToSend}("");
require(success, "Call failed");
```
