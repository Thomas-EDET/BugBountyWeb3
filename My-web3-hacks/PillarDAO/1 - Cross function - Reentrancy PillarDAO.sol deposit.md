Cross-function Reentrancy attack on PillarDAO.sol contract.

Deposit() exploitation:

![deposit-function](https://github.com/user-attachments/assets/c7d02549-486e-4836-9901-53deee5448d7)

We can see that it does not respect Check Effect Interaction principle. 

Token.safeTransferFrom is called before making the changes which are:<br> 
-memberships[msg.sender]  = membershipNFT.mint(msg.sender)<br> 
-balances[msg.sender] = Deposit({<...>})<br> 

Potential impact:
The attacker could've potentially mint multiple NFTs with only one deposit. This would be a form of duplication exploit where they get multiple membership NFTs for the price of one.

### Exploitation

#### Entry point:<br> 
Slither found 2 external calls that can be used for reentrancy attack<br> 
token.safeTransferFrom(msg.sender, address(this), _amount);
memberships[msg.sender] = membershipNFT.mint(msg.sender);


#### Attack flow for MembershipNFT.mint:
I decided to chose membershipNFT.mint() only because I already exploited safeTransfer function in the withdraw() function earlier. I like changes.

![flow-deposit](https://github.com/user-attachments/assets/92096224-6b46-4053-af02-9c1fdb2aa33e)



#### Technical analysis:
membershipNFT.mint(msg.sender); is making use of mint a function  from membershipNFT from membershipNFT.sol:

`function mint(`<br> 
`address _to`<br> 
`) external onlyVault nonReentrant returns (uint256) {`<br> 
`uint256 mintIndex = _tokenIds;`<br> 
`_safeMint(_to, mintIndex);`<br> 
`++_tokenIds;`<br> 
`emit Minted(_to, mintIndex);`<br> 
`return mintIndex;`<br> 
`}`<br> 

Here we can  see this function is  making  use of _safeMint  which is from ERC721Enumerable.sol as 

`contract MembershipNFT is **ERC721Enumerable**, Ownable, ReentrancyGuard {`<br> 
`using SafeERC20 for IERC20;`

ERC721Enumerable.sol->
`abstract contract ERC721Enumerable is ERC721, IERC721Enumerable {`<br> 

So we check ERC7210.sol from OZ:


`function _safeMint(address to, uint256 tokenId) internal virtual {`<br> 
`_safeMint(to, tokenId, "");`<br> 
`}`<br> 
<br> 
`function _safeMint(`<br> 
`address to,`<br> 
`uint256 tokenId,`<br> 
`bytes memory _data`<br> 
`) internal virtual {`<br> 
`_mint(to, tokenId);`<br> 
`require(`<br> 
`_checkOnERC721Received(address(0), to, tokenId, _data),`<br> 
`"ERC721: transfer to non ERC721Receiver implementer"`<br> 
`);`<br> 
`}`<br> 

We can see two functions with the same name _safeMint, when people  call the first safeMint function it automatically call the second one,  the first one is a wrapper for the second one and it triggers a callback because we have _checkOnERC721Received and  if we check what this function is  doing:

`function _checkOnERC721Received(`<br> 
`address from,`<br> 
`address to,`<br> 
`uint256 tokenId,`<br> 
`bytes memory _data`<br> 
`) private returns (bool) {`<br> 
`if (to.isContract()) {`<br> 
`try IERC721Receiver(to).onERC721Received(_msgSender(), from, tokenId, _data) returns (bytes4 retval) {`<br> 
`return retval == IERC721Receiver.onERC721Received.selector;`<br> 
`} catch (bytes memory reason) {`<br> 
`if (reason.length == 0) {`<br> 
`revert("ERC721: transfer to non ERC721Receiver implementer");`<br> 
`} else {`<br> 
`assembly {`<br> 
`revert(add(32, reason), mload(reason))`<br> 
`}`<br> 
`}`<br> 
`}`<br> 
`} else {`<br> 
`return true;`<br> 
`}`<br> 
`}`<br> 

We can see that it tries to fetch on ERC721Received() if the recipient is a contract. This function need to be created by us so this is our callback. Because of that we can trigger other vulnerable function from PillarDAO.sol at least it's what I believed.

#### Proof Of Concept

A malicious contract could implement `onERC721Received()` to call back into the DAO's functions while `deposit()` is still running, potentially exploiting the fact that `memberships[msg.sender]` is updated but `balances[msg.sender]` is not yet.

Here is the function onERC721Received() I made that we can use to verify we can use the reentrancy vulnerability:
![fallback-deposit](https://github.com/user-attachments/assets/b2485993-de9a-4c3b-a143-b09a208cd8b7)


We can see we are not executing any other function only getting few variables and print them.

After running our test obviously the comparison we made is false as OnERC721Received doesn't implement any malicious function to reenter the contract:
![test-run-deposit-reentrancy](https://github.com/user-attachments/assets/9a843d5b-1367-494f-9748-3e6b27215b42)


#### Conclusion

We proved that reentrancy worked as during the callback we can see the deposit is not made but we got a membership ! This is because state variables are updated after the interaction  and not before, this is not respecting the Check-Effect-Interaction principle.

if we had something like this : function vote(uint proposalId, bool support) external {
    require(memberships[msg.sender] > 0, "Must be a member");
    // No check for balances[msg.sender]
    // Count vote...
}

**We could've exploited this function allowing potentially one member to vote two times: One time during the cross function reentrancy attack and potentially another time after submitting our token, that is  needed to complete the membership enrollment. In practice, this means the contract has a vulnerability that's waiting for the right conditions (like a new vote function being added) to become exploitable.**


#### Bonus

Later on I tried to call withdraw() in our callback but I forgot it would obviously not worked as :
balances[attacker].depositAmount is still 0 (since state are not updated)
And balances[attacker].depositTime is not set yet as to withdraw a user need to  wait  52 weeks.

See function here:

function withdraw() external override  {<br> 
        require(<br> 
            balances[msg.sender].depositAmount > 0,<br> 
            "PillarDAO:: insufficient balance to withdraw"<br> 
        );<br> 
        require(<br> 
            (block.timestamp - balances[msg.sender].depositTime) > stakingTerm,<br> 
            "PillarDAO:: too early to withdraw"<br> 
        );<br> 
        require(<br> 
            memberships[msg.sender] > 0,<br> 
            "PillarDAO:: membership does not exists!"<br> 
        );<br> 
