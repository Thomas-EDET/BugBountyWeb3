A DeFi protocol has a lending system where users can deposit WETH as collateral and borrow DAI. The protocol uses Chainlink for price feeds, but has a fallback TWAP mechanism from a Uniswap V2 pool if Chainlink fails. A researcher notices the following:

- The Uniswap TWAP uses a 10-minute window.
- The protocol checks for Chainlink oracle staleness (over 30 minutes) before falling back.
- Borrow limit is 75% LTV.
- Liquidations are triggered at 85% LTV.
- There’s no cap on how much DAI a user can borrow per block.

**You have 100 ETH and 10K DAI. Design a profitable attack path, and explain each key step clearly.**



<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>
Answer:<br>
1 -  Crash the chainlink price feeds or wait for the platform to switch for TWAP<br>
2 - Buy a bunch of ETH so ETH price will increase, this action will only be possible in low liquidity pool. Wait for the price to increase<br>
3 - Buy a bunch of DAI with your overvalued collateral. <br>
4 - Now you've more DAI, walk away from the platform and enjoy your +PnL<br>


TWAP calculation example:<br>
- If ETH is $3,000 and you manipulate it to $6,000 for 1 minute,<br>
- And the TWAP window is 10 minutes,<br>
- The new TWAP will be:  <br>
    `((9 min * $3000) + (1 min * $6000)) / 10 min = $3300`<br>
