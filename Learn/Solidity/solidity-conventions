- Convention and Readability: Many developers follow a typical organizational pattern in Solidity:

- State variables first
- Events next
- Modifiers next
- Constructor
- Functions (often grouped by visibility/purpose)


_ before the function is a convention that stipulate the function is supposed to  be called internally. 
`function investEther() public isOpen whenNotPaused payable {`
`_investEther(_msgSender(), msg.value);`
`_rebalance();`
`}`

We can also see that the user can  call this function that has not parameters but  send a parameter in _investEther, it's because the fonction is payable. Therefore msg.value is the amount of eth in wei.

By calling this function and sending eth, it automatically invest the ether and rebalance.
