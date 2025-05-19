Implementation contract:

```solidity
contract VaultLogic {
    address public owner;  // <-- slot 0
    mapping(address => uint256) public balance;

    function deposit() external payable {
        balance[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 amount = balance[msg.sender];
        require(amount > 0, "Nothing to withdraw");
        balance[msg.sender] = 0;
        payable(msg.sender).transfer(amount);
    }
}
```
```solidity
Proxy contract
contract StorageProxy {
    address public implementation;

    function upgrade(address newImplementation) external {
        implementation = newImplementation;
    }

    fallback() external payable {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```


There is no vulnerabiity in those contract  but later if  the owner  decides  to  upgrade the implementation  with:

```solidity
function initialize(address _owner) external {
    require(owner == address(0), "Already initialized");
    owner = _owner;
}
```
<br>
Then an attacker can call initialize and overwrite the implementation contract address since implementation address is set to 0.<br>

> The **proxy contract** delegates execution to the logic contract, but it still uses **its own storage**.<br>

That’s the core of `delegatecall`<br>

To target `slot N`, you need a function in the logic contract that assigns to the `N`th declared variable.<br>
