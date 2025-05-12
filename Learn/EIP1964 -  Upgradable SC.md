EIP-1967, **transparent proxy pattern** 

**Proxy Contract**: This is the address users interact with (the front-facing contract)

**Implementation Contract**: This contains the actual logic but users don't interact with it directly
**Storage**: The proxy owns and maintains the storage, while the implementation provides the logic

ABI shows only the functions of the proxy contract and not the implementation contract.

Therefore if we want to test the logic that is implemented by the implementation contract we need to use the proxy contract address and the function found in read as proxy of this proxy contract.
