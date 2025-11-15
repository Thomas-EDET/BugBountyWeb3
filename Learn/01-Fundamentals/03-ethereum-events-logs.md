While analyzing Venice contracts I found out  that you have a section on baseexplorer called "Event".

I was expecting the name of the function to be TransferFrom rather than Transfer as ERC20 specifies that Transfer() is only two parameter : to and amount while we go three here: from, to, amount.

The reason:
- Both `transfer()` and `transferFrom()` **emit the same** `Transfer` event
- Ethereum **logs events**, not functions

To know **which function was called**, you’d have to:
- Look at the **transaction input data** (the calldata)
- Decode the **function selector** (first 4 bytes of calldata)
