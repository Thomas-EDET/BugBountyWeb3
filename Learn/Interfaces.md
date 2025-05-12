1  - Introduction:
Like API endpoints but for contracts.

Ex Web2: 
v1/api/spaceweapons/func1
v1/api/spaceweapons/func2
v1/api/orgname/func1
v1/api/orgname/func2
v1/api/orgname/func3

Web3 example with  AAVE:
Aave/smartcontracts
Project/smartcontracts/Pool/
Project/smartcontracts/Pool/supply -> supply  the protocol with cryptocurrencies

supply  is described as follow:
function supply( address asset, uint256 amount, address onBehalfOf, uint16 referralCode) public virtual override

2 -  implementation:
First we need to need to be implemented like this :
interface ILendingPool {

function deposit(
address asset,
uint256 amount,
address onBehalfOf,
uint16 referralCode
) external;

function borrow(
address asset,
uint256 amount,
uint256 interestRateMode,
uint16 referralCode,
address onBehalfOf
) external;
}

->  here we can see that we are using two functions that are present in the pool  smart contract of aave. 

3  - Using the interface:

contract AaveWrapper is Ownable, ReentrancyGuard {
ILendingPool public lendingPool;

constructor(
address _lendingPool,
<...>
) Ownable(msg.sender) {
require(_lendingPool != address(0), "Invalid lending pool address");

lendingPool = ILendingPool(_lendingPool);

}

3 - Deploying the contract that makes use of the interface:

contract DeployAaveWrapper is Script {
function run() external {
// Get the deployer's address
address deployer = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266;
// Set up deployment parameters
address AAVE_LENDING_POOL = 0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2; //HERE  is the lending pool address it can't be empty as specified in the constructor.
uint256 INITIAL_FEE_RATE = 100; // 1%
address FEE_COLLECTOR = deployer;
address WETH_ADDRESS = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address USDC_ADDRESS = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
// Uniswap V3 SwapRouter address for Ethereum mainnet
address UNISWAP_ROUTER = 0xE592427A0AEce92De3Edee1F18E0157C05861564; // Uniswap V3 SwapRouter

// Start broadcasting with prank to ensure we're using the funded account
vm.startBroadcast();
AaveWrapper wrapper = new AaveWrapper(
AAVE_LENDING_POOL,
INITIAL_FEE_RATE,
FEE_COLLECTOR,
WETH_ADDRESS,
USDC_ADDRESS,
UNISWAP_ROUTER
);
vm.stopBroadcast();

// Log the deployed address and final balance
console.log("AaveWrapper deployed at:", address(wrapper));
console.log("Final deployer balance (wei):", deployer.balance);
}
}
