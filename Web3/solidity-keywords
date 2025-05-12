Part of test.sol  from forge-std - testing purposes:
vm.prank(user1); -> set the next transaction's sender to user1 (when we launched anvil)
vm.warp(block.timestamp + 15 seconds); -> play with block timestamp useful for casino dApp
vm.expectRevert("Only the vault can mint or burn");  -> if we are expecting a failure with  a specific message
vm.prank(address); // Impersonate an address for the next call vm.startPrank(address); // Impersonate an address for multiple calls vm.stopPrank(); // Stop impersonation
vm.deal(address, amount);
assertEq(nft.balanceOf(user1), 1); -> check if two  values are equal
assertTrue(longTotalFee <= shortTotalFee, "Long fee should be lower than or equal to short fee");  ->  Verifies that a condition evaluates to true

Solidity keywords:
require -> require info to  continue
payable is a function attribute if we want the contract to receive eth
public -> accessible from internal  and  external
external -> function only accessible from external call
internal -> function callable only  from the contract  itself or other owned contracts
private ->  only accessible from the contract itself
override ->  override a parent function is import
super. -> use a parent function in import
pure  -> assure the function will not write anything
tx.origin -> dangerous function that can be abused to bypass  authentication by using physhing style attack
fallback() -> executed when a contract call provides data that doesn’t match any function signature **or** when sending ETH without a `receive()`.
receive() - >  executed when the contract receives plain ETH (no calldata).
event Minted(address indexed owner, uint256 indexed tokenId);  -> used for logs  purpose, write parameters on the chain and can  be queried whenever we want whatever parameters we want. Useful for displaying  on UI or for debug.
Emit() -> Events are triggered/fired using the `emit` keyword
transient  -> Reentrancy locks, Single transaction [ERC-20](https://eips.ethereum.org/EIPS/eip-20) approvals
