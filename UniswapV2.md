## UniSwapV3 Notes

### AMM(Automated Market Maker):
1. All the type of token are on the curve we have to define
2. we can calculate price ratio of any token by the help of the curve with respect to the time with change in the market.
3. All the thing depend on the curve of the AMM


### UniSwapV2 Contract are divided in three parts:
1. Factory contract: It is simple contract that deploy simple contract
2. Pair Contract: Pair contract is the contract that hold the pair of ERC20 Token to give the exchange of both.For Example: we can swap ERC20 USDC token with ETH token
3. Router contract : This contract main work to remove the liquidity and add liquidity from the ERC20 token with the help of contract (Liquidity : That provided by external people to get gas fee and make there profit that we have already learn in TSwap contract) and SWap function valid in Router contract with the help of factory contract and then factory contract call pair contract this so on happened.
