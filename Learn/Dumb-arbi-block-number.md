**[HELP  Arbitrum forking via vm.createSelectFork]**[SOLVED]

Good morning everyone, I'm not succeeding in forking the latest  block number for arbitrum through a solidity  test  via vm.createSelectFork,  I tried multiple things including:
`Changing the RPC provider
Cleaning the foundry/forge cache
Setting up fresh projects without historical cache
Manually specifying full URLs in the code
Using direct block number with default parameters
Hexadecimal notation (0x13ee2390)Hexadecimal notation (0x13ee2390)
Double checked if I  wasn't forking sepolia arbitrum chain`

What's fun is when I use anvil -f ... --fork-block-number 334815142  it seems to  work correctly :
Fork
==================
Endpoint:       https://arb-mainnet.g.alchemy.com/xxxx
Block number:   334815142
Block hash:     0x40478b2d88d934462b18ccc8b375233df0cfb0df2f58e82b84adde7249ef97cd
Chain ID:       42161

**But actually when I check the block number using assert it shows an  incorrect block number,** same error that I  had before using vm.createselectFork:
[FAIL: Wrong block number: 22444608 != 334815142] 

If anyone got a similar issues with Arbitrum forking I  would much appreciate their help ! 

**__-> solution: __**
After using vm.createselectfork() and trying to display the block number  using console.log(block.number) it displays the block number of L1 not L2.
To find the actual L2 block number that has been forked use:  cast block latest --rpc-url http://localhost:8545
