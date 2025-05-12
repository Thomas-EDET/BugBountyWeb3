_GSN abstracts away gas_ to minimize onboarding & UX friction for DApps. It is a decentralized system that improves DApp usability without sacrificing security.

`abstract contract Context {`
`function _msgSender() internal view virtual returns (address) {`
`return msg.sender;`
`}`
`function _msgData() internal view virtual returns (bytes calldata) {`
`return msg.data;`
`}`
`}`

In Mt Pelerin audit:
receive () external isOpen whenNotPaused payable {
    require(msg.data.length == 0, "TS08");
    _investEther(_msgSender(), msg.value);
    _rebalance();
  }

  function investEther() public isOpen whenNotPaused payable {
    _investEther(_msgSender(), msg.value);
    _rebalance();
  }

we can see they are making use of _msgSender() which is an inherited function from OZ context.sol. This allows to  modify the msg.sender which can be handy for gas fee (if someone wants to pay for the gas for the user .)


There are several advantages to using _msgSender() over directly using msg.sender:

- Meta-transaction Support: It allows contracts to support meta-transactions where someone else might pay for gas
- Upgradability: Makes it easier to change how the sender is determined in future versions
- Gas Station Network (GSN): Enables gasless transactions for users
