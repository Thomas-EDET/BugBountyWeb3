Reentrancy attack on PillarDAO.sol contract.


My first reentrancy finding in real world contract:

![[Screenshot from 2025-04-25 14-13-32.png]]

We can see that it does not respect Check Effect Interaction. 

Token.safeTransferFrom is called before making the changes which are:
-memberships[msg.sender]  = membershipNFT.mint(msg.sender)
-balances[msg.sender] = Deposit({<...>})

Why it is mitigated:
We have a require statement  : `require(memberships[msg.sender] == 0, "PillarDAO:: user is already a member");`

Once we reenter the function check if the member is already a member and yes we are so then it stops here.  with error  PillarDAO::: user is already a  member.

Potential impact:
The attacker could've potentially mint multiple NFTs with only one deposit. This would be a form of duplication exploit where they get multiple membership NFTs for the price of one.



Exploitation of **membershipNFT.mint(msg.sender);**
token.safeTransferFrom(msg.sender, address(this), _amount);
memberships[msg.sender] = membershipNFT.mint(msg.sender);

**membershipNFT.mint(msg.sender);** is the second call that can be exploited for a reentrancy and we  will try to exploit it, when reentering into a function the last exploitable function is often the better as we  skip all previous instructions that could blocked us.

membershipNFT.mint(msg.sender); is making use of mint a function  from membershipNFT from membershipNFT.sol:

`function mint(`
`address _to`
`) external onlyVault nonReentrant returns (uint256) {`
`uint256 mintIndex = _tokenIds;`
`_safeMint(_to, mintIndex);`
`++_tokenIds;`
`emit Minted(_to, mintIndex);`
`return mintIndex;`
`}`

Here we can  see this function is  making  use of _safeMint  which is from ERC721Enumerable.sol as 

`contract MembershipNFT is **ERC721Enumerable**, Ownable, ReentrancyGuard {`
`using SafeERC20 for IERC20;`

ERC721Enumerable.sol->
`abstract contract ERC721Enumerable is ERC721, IERC721Enumerable {`

So we check ERC7210.sol from OZ:


`function _safeMint(address to, uint256 tokenId) internal virtual {`
`_safeMint(to, tokenId, "");`
`}`

`function _safeMint(`
`address to,`
`uint256 tokenId,`
`bytes memory _data`
`) internal virtual {`
`_mint(to, tokenId);`
`require(`
`_checkOnERC721Received(address(0), to, tokenId, _data),`
`"ERC721: transfer to non ERC721Receiver implementer"`
`);`
`}`

We can see two functions with the same name _safeMint, when people  call the first safeMint function it automatically call the second one,  the first one is a wrapper for the second one and it triggers a callback because we have _checkOnERC721Received and  if we check what this function is  doing:

`function _checkOnERC721Received(`
`address from,`
`address to,`
`uint256 tokenId,`
`bytes memory _data`
`) private returns (bool) {`
`if (to.isContract()) {`
`try IERC721Receiver(to).onERC721Received(_msgSender(), from, tokenId, _data) returns (bytes4 retval) {`
`return retval == IERC721Receiver.onERC721Received.selector;`
`} catch (bytes memory reason) {`
`if (reason.length == 0) {`
`revert("ERC721: transfer to non ERC721Receiver implementer");`
`} else {`
`assembly {`
`revert(add(32, reason), mload(reason))`
`}`
`}`
`}`
`} else {`
`return true;`
`}`
`}`

We can see that it tries to fetch on ERC721Received() if the recipient is a contract. This function need to be created by us so this is our callback. Because of that we can trigger other vulnerable function from PillarDAO.sol at least it's what I believed.

To sum up here is our attack flow : 
MembershipNFT.mint()
  ↓
  _safeMint() // OZ Implementation
    ↓
    _mint()    // Updates internal state
    ↓
    _checkOnERC721Received() // Callback mechanism
      ↓
      recipient.onERC721Received() // Control given to recipient
        ↓
        [ATTACK HAPPENS HERE] // Recipient can call back into PillarDAO
          ↓
          PillarDAO.withdraw() or other functions // Before deposit() finished updating state
            ↓
            [EXPLOIT STATE INCONSISTENCY]



So here is our implementation of onERC721Received  that allows the execution of code  from our side:
`function onERC721Received(`
`address,`
`address,`
`uint256 tokenId,`
`bytes calldata`
`) external returns (bytes4) {`
`// Prevent recursive calls`
`if (!alreadyAttacked) {`
`alreadyAttacked = true;`
`console.log("ATTACK: Inside onERC721Received callback!");`
`console.log("ATTACK: Membership ID at this point:", tokenId);`
`console.log("ATTACK: Deposit amount at this point:", dao.balanceOf(address(this)));`
`}`
`// Must return this to avoid reverting the mint`
`return this.onERC721Received.selector;`
`}`

We can see we are not executing any other function only getting few variables and print them.

Claude:
A malicious contract could implement `onERC721Received()` to call back into the DAO's functions while `deposit()` is still running, potentially exploiting the fact that `memberships[msg.sender]` is updated but `balances[msg.sender]` is not yet.

After running our test obviously the comparison we made is false as OnERC721Received doesn't implement any malicious function to reenter the contract:
Ran 1 test for test/PillarDAO.t.sol:PillarDAOTest
[FAIL: Should still have original tokens: 0 != 10000000000000000000000] testNFTCallbackReentrancyExploit() (gas: 4561835)
Logs:
  ---- Initial State ----
  Attacker token balance: 10000000000000000000000
  Attacker membership ID: 0
  Attacker deposit amount: 0
  ATTACK: Inside onERC721Received callback!
  ATTACK: Membership ID at this point: 1
  ATTACK: Deposit amount at this point: 0
  ---- Final State ----
  Attacker token balance: 0
  Attacker membership ID: 1
  Attacker deposit amount: 10000000000000000000000

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 3.33ms (1.07ms CPU time)

Ran 1 test suite in 9.88ms (3.33ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/PillarDAO.t.sol:PillarDAOTest
[FAIL: Should still have original tokens: 0 != 10000000000000000000000] testNFTCallbackReentrancyExploit() (gas: 4561835)


So we end up submitting our token and getting  a membership with nothing happening incorrectly. 

However we proved that reentrancy worked as during the callback  we can see the deposit is not made but we got a membership  ! This is because  state variables are updated after the interaction  and not before not respecting the Check-Effect-Interaction principle.

This is because despite the state inconsistency, there's no function in the current PillarDAO that we can exploit during this window to prevent the deposit from completing.

if we had something like this : function vote(uint proposalId, bool support) external {
    require(memberships[msg.sender] > 0, "Must be a member");
    // No check for balances[msg.sender]
    // Count vote...
}

we  could've exploited this function allowing potentially one member to vote two times: once before submitting his token and once after submitting his token if not check function is present is vote()

Later on I tried to  call withdraw() in our callback but I forgot it would obviously not worked as :
balances[attacker].depositAmount is still 0 (since state are not updated)
And balances[attacker].depositTime is not set yet as to withdraw a user need to  wait  52 weeks.

