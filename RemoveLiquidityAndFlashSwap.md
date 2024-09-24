# Remove Liquidity and Flash Swap

## Graph of Removing Liquidity from AMM:
When we add liquidity to a constant product AMM, the amount of token X and the amount of token Y just follow a simple rule. The price after adding liquidity must be equal to the price before adding liquidity. Again same rule is followed in removing liquidity from AMM .
- price after add liquidity = price before
    - `y + dy / x + dx = y / x`

## Remove Liquidity
- How many dx and dy to withdraw ?
    - s = shares to burn , T = total shares, L0 = liquidity before, L1 = Liquidity after
      - ` L0 - L1 = s/T * L0 ` and `L0 - L1 / L0 = dx/x = dy/y` by these two equation, `dx = x * s/T` and `dy = s / T * y`
      - - f(x,y) = sqrt(x*y)
        - Motivation:
            - x * y = L ^ 2 --> sqrt(x*y) = L
            - L0 - L1 / L0 = f(x,y) - f(x-dx,y-dy) / f(x,y)
                           => sqrt((x) * (y)) - sqrt((x - dx)*(y - dy))/ sqrt(x * y)  by dy = y/x * dx
                           we get, `L0 - L1 / L0 = dx/x` so, `s/T = dx/x`

## Remove Liquidity (calls):
1. `user` call Router contract to `removeLiquidity`
2. `transferFrom` LP to `DAI/ETH` pair 
3. `burn` is call by the router contract burn the `DAI/ETH` 
4. `transfer` the token to `user`


```javascript
// **** REMOVE LIQUIDITY ****
    function removeLiquidity(
        address tokenA,// WETH
        address tokenB,// DAI
        uint liquidity,// liquidity 
        uint amountAMin, // just minimum amount
        uint amountBMin,
        address to, // user
        uint deadline // block.timestamp
    ) public virtual override ensure(deadline) returns (uint amountA, uint amountB) {
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        // pair contract address 
        IUniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity); // send liquidity to user , if pair contract does not exists it fails and does not execute 
        (uint amount0, uint amount1) = IUniswapV2Pair(pair).burn(to);// burning liquidity from pair of token
        (address token0,) = UniswapV2Library.sortTokens(tokenA, tokenB); // sort them 
        (amountA, amountB) = tokenA == token0 ? (amount0, amount1) : (amount1, amount0);
        require(amountA >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
        require(amountB >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
    }

```

### Testing Remove Liquidity

```javascript

    function test_removeLiquidity() public {

        ERC20 DAI = new ERC20("dai","DAI",18);
        ERC20 WETH = new ERC20("weth","WETH",18);

        address user = makeAddr("USER");

        vm.startPrank(user);

        (,,uint256 liquidity) = router.addLiquidity({
            tokenA : DAI,
            tokenB: WETH,
            liquidity: liquidity,
            amountADesired: 1000000 * 1e18,
            amountBDesired: 100 * 1e18,
            amountAMin: 1,
            amountBMin: 1,
            to: user,
            deadLine: block.timestamp()
        
        });

        pair.approve(address(router),liquidity);

        (uint amountA,uint amountB) = router.removeLiquidity(tokenA,tokenB,liquidity,amountAMin,amountBMin,to,deadline);

        vm.stopPrank();

        console2.log("DAI",amountA);
        console2.log("WETH",amountB);

        assertEq(pair.balanceOf(user),0,"LP = 0");

    }

```


## Flash Swap Fee:


-> Swap dx for dy :
    - amount in = dx
    - amount in - fee = 0.997 * dx
    - amount out = dy
    - `swap fee = 0.003 * dx`

-> Flash Swap 
    - amount out = borrow -> dx
    - amount in = repay,dx1 = dx + fee 
    - amount in - fee = 0.997 * dx1
    - swap fee = 0.003 * dx1

### Flash Swap Fee Equation
- `x - dx + 0.997 * dx1 >= x ` , dx: amount out, 0.997 * dx1 = amount In - fee , x = reserve before
- Here we are comparing , `after reserve >= before reserve`
- `dx1 = dx + fee `, by solving both equation
- we get , `fee >= 3 / 997 * dx`


## Flash Swap fee (Calls):
1. `user` start flash swap .
2. The flash Swap contract `swap` function on the pair contract.
3. The contract send requested amount of tokens to the flash swap contract simultaneously.
4. The flash swap contract can then perform any custom logic using the borrowed token (like attacking to any other contract and arbitrage). This is done by `uniswapV2Call` function inside the flash loan swap.
5. THe pair contract verifies that the flash swap / loan has been repaid the borrowed token and fees.

```
graph LR
    user["User"] --> flash_swap["Flash Swap"]
    flash_swap --> pair["Uniswap V2 Pair Contract"]
    pair --> flash_swap
    flash_swap --> custom_logic["Custom Logic"]
    flash_swap --> uniswap_call["uniswapV2Call"]
    uniswap_call --> pair
    subgraph "Tokens"
        WETH["WETH"]
        DAI["DAI"]
    end
```

### Flash Swap function:

```javascript

contract UniswapV2FlashSwap {
    UniswapV2Pair private immutable pair;
    address private immutable token0;
    address private immutable token1;

    constructor(address _pair) {
        pair = UniswapV2Pair(_pair);
        token0 = pair.token0();
        token1 = pair.token1();
    }


    function flashLoan(address token, uint256 amount) external {
        require(token == token0 || token == token1, "invalid token");


        (uint256 amount0Out, uint256 amount1Out) = token == token0 ? (amount,uint256(0)):(uint256(0),amount);

        bytes memory data = abi.encode(token,msg.sender);

        pair.swap(amount0Out,amount1Out,address(this),data);

    }


    function uniswapV2Call(address sender,uint256 amount0, uint256 amount1,bytes calldata data) external {
        // require msg.sender is pair contract because no external call uniswapV2Call because they can transfer all assets
        // require sender is  this contract
        require(msg.sender ==  address(pair),"not pair");
        require(sender == address(this),"not sender");

        // Decode token and caller from data;
        (address token, address caller) = abi.decode(data,(address,address));
        // determine amount borrowed (only one of them > 0)
        uint256 amount = token == token0 ? amount0 : amount1;

        // calculate the fee for flash swap 
        uint256 fee = ((3 / 997) + 1) * amount ;

        uint256 amountToRepay = amount + fee ;

        // Get flash swap fee from caller
        IERC20(token).transferFrom(caller, address(this),fee);
        // Repay Uniswap V2 pair
        IERC20(token).transfer(address(pair),amountToRepay);


    } 
}


```