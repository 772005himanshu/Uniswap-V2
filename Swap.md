# SWAP

Ex : Alice Swap dx for dy
-> x + dx and y - dy because we have to maintain the proportionality ( x * y = constant (This is also called the invariant))

Find dy:
   - Before swap:
    x*y = L^2

   - After swap
   - (x + dx)*(y - dy) = L^2
- comparing both equation,
   - x * y = (x + dx)*(y - dy)
   - x * y / (x + dx) + dy = y
   - dy = y - x * y / (x + dx)
   - `dy = y * dx / x + dx  -> dy` without fee
  
## Swap Fee
Alice swaps dx for dy
Swap fee charged on token in 
- 0 <= f <= 1 , swap fee = fdx
- we have to replace dx -> (1-f) * dx
- dy with fee
   dy = dx * y / x + dx -> without fee equation
   `dy = (1-f) * dx * y / (x + (1-f) * dx)`


## Swap Contract call
- Call to swap ERC20 token 
- By the Help Router contract and factory contract we can swap the token 
  1. user can call too many function from the contract . For swaping the function we are calling is the `swapExactTokensForTokens` . If the token is not swap from ERC20 to ERC20 then we have to call for another function or write the function for token to ERC20 token function. 
  2. user Call for `transferFrom` function transfer the amount.
  3. Then user call for `Swap` and `transfer` the another token to user.
  4. What is the difference between `transferFrom` and `transfer` .transferFrom function take to, amount only where as transfer function from,to,amount.


## Code -> Swap 
1. The code for the Uniswap V2 protocol is split in two part core and perhiperry contract .V2 periphery contains the router contract that is used to ad and remove the liquidity and swaps also
2. The actual contract that handles liquidity and swaps of tokens is in V2 core and the main contract is `Uniswap V2 Pair` contracts.
3. Why there is two type of contract main reason is `Uniswap V2 Pair` is used for pair and swap of token .But router contact is used for multiple swap and is used for utility . This is used minimize the lose of funds by pair of wrong token because we cannot reverse that process.
4. The `swapExactTokensForTokens` function first transfers the input token to the pair contract and The pair contract is calculated using a function called `create2` based on input and output token address provided in the `path` array
5. inside the `Swap` function .the function loops through the `path` array ,iterating from index `zero` to `path.length - 1`


## Swap -> getAmountsOut();

```javascript

  function getAmountsOut(address factory,uint amountIn,address[] memory path) interval view returns(uint){
   require(path.length >= 2, "UniswapV2Library: INVALID_PATH");
   amounts = new uint[](path.length); // why we are not using uint[] amounts(path.length);
   amounts[0] = amountsIn;

   for(uint i;i<path.length-1;i++){
      // reserves = internal balance of tokens inside pair contract
      (uint reservesIn, uint reservesOut) = getReserves(factory,path[i],path[i+1]);
      // use the previous output for input
      amounts[i+1] = getAmountOut(amounts,reserveIn,reserveOut);
   }

  }

  function getAmountOut(uint amountIn, uint reserveIn,uint reserveOut) internal pure returns(uint amountOut){
   require(amountIn > 0 && reserveIn > 0 && reserveOut > 0 , "UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT");
   // `dy = (1-f) * dx * y / (x + (1-f) * dx)` with fee 
   // reserveOut  = y
   // reserveIn = x
   // AmountIn = dx
   // AmountOut = dy
   // fee of the protocol is 3% of the input amount

   uint amountInWithFee = amountIn.mul(997);
   uint numerator = amountInWithFee.mul(reserveOut);
   uint denominator = reserveIn.mul(1000).add(amountInWithFee);
   amountOut = numerator / denominator;
  }

```


Testing the contract:

```javascript
   // SPDX-License-Identifier: MIT
   pragma solidity ^0.8.0;
   import {Test,console2} from "forge-std/Test.sol";
   import "./interfaces/IERC20.sol";
   import "./interfaces/IUniswapV2Router02.sol";
   import "./uniswapV2Router02.sol";

   contract UniswapV2SwapAmountsTest is Test {
      IERC20 private constant weth = IERC20(WETH);
      IERC20 private constant dai = IERC20(DAI);
      IERC20 private constant mkdr = IERC20(MKR);

      IUniswapV2Router02 private constant router = IUniswapV2Router02(UNISWAP_V2_ROUTER_02);

      function test_getAmountsOut() public {
         // getAmountsOut(uint amountIn, address[] memory path) returns(uint amounts)
         address[] memory path = new address[](3);

         path[0] = WETH;
         path[1] = DAI;
         path[2] = MKR;

         uint amountIn = 1e18;
         uint[] memory amounts = router.getAmountsOut(amountIn,path);

         console2.log("WETH",amounts[0]);
         console2.log("DAI",amounts[1]);
         console2.log("MKR",amounts[2]);
      }
   }

```

## Function swapTokensForExactTokens:

```javascript
   function swapTokensForExactToken(uint amountOut,uint amountInMax, address[] calldata path,address to,uint deadline) external override virtual ensure(deadline) returns(uint[] memory amounts){
      amounts = UniswapV2Library.getAmountIn(factory,amountOut,path);
      require(amount[0] <= amountInMax, "UniswapV2Router: EXCESSIVE_INPUT_AMOUNT");
      TransferHelper.safeTransferFrom(path[0],msg.sender,UniswapV2Library.Pair(factory,path[0],path[1],amounts[0]));
      _swap(amounts,path,to);
   }


```

Explanation of upper code:`amountOut` is from `getAmountOut` function it is like input from another function we are doing here,we are storing amountOut in `amounts` array type we are calling from here . We are here directly transfer token to another Token Then we are pairing token together and last to be swap.

## function getAmountIn():

```javascript
   function getAmountIn(uint amountOut, uint reserveIn, uint reserveIn) internal pure return(uint amountIn){
      require(amountOut > 0, "UniswapV2Library: INSUFFICIENT_OUTPUT_AMOUNT");

      require(reserveIn > 0 && reserveOut > 0 , "UniswapV2Library: INSUFFICIENT_LIQUIDITY");
      // The formula we have driven earlier dy = y * (1-f)*dx/x + y * dx(1-f)
      // by this we have to calculate dx = x * dy / (y - dy) * (1-f)
      uint numerator = reserveIn.mul(amountOut).mul(1000);
      uint denominator = reserveOut.sub(amountOut).mul(997);

      amountIn = (numerator / denominator).add(1); // adding one for just Rounding-up
   }


   function getAmountsIn(address factory, uint amountOut,address[] memory path) internal view returns(uint[] memory amounts){
      require(path.length >= 2,"UniswapV2Library: INVALID_PATH");

      amounts = new uint[](path.length);
      amounts[amounts.length -1] = amountOut;


      for(uint i = path.length - 1;i>0;i--){
         (uint reserveIn,uint reserveOut) = getReserves(factory.path[i-1],path[i]);
         amounts[i-1] = getAmountIn(amounts[i],reserveIn,reserveOut);
      }
   }

```

Testing GetAmountsIn function:

```javascript

   function test_getAmountIn() public {
      address[] memory path = new address[](3);

      path[0] = WETH;
      path[1] = DAI;
      path[2] = MKR;
      uint amountOut = 1e18;

      uint[] memory amounts = router.getAmountsIn(amountOut,path);

      console2.log("WETH",amounts[0]);
      console2.log("DAI",amounts[1]);
      console2.log("MKR",amounts[2]);

   }

```

## Swap-Pair contracts:

```javascript
   
   function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
        // checking amountOut of both token should be greater than 0;

        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        // reserve amount should be taken from getReserves function
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');
        // it is necessary that amountOut should be less than reserve amount in the contract


        // after and before  swap  accounts balance and after swap we have to update the balance of the contract
        uint balance0;
        uint balance1;
        { // scope for _token{0,1}, avoids stack too deep errors
        address _token0 = token0;
        address _token1 = token1;
        require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO'); // my thinking on this is it has two method of working (lock and unlock) and (burn and mint) function working that why we are not transfer token to anu account just pushing to the contract or function
        if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
        if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
        if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data); // we are just checking the data through UniswapV2Call that work with data and value if another contract
        balance0 = IERC20(_token0).balanceOf(address(this)); // balance Checking
        balance1 = IERC20(_token1).balanceOf(address(this));
        }
        // This should be clear by example
        // amount0out = 0;
        // amount1out = 10;
        // amount in = 10 token 0 // we have already transfer  10 token to swap
        // balance0 = 1010;
        // _reserve0 = 1000 ;

        //               1010 > 1000 - 0                --> 1010 - 1000 - 0 == 10
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        { // scope for reserve{0,1}Adjusted, avoids stack too deep errors
        // NOTE:
        // amount0In = 0 -> balance0Adjusted = balance0;
        // amount0In > 0  -> balance0Adjusted = balance0 * 1000 - 3 * amount0In
        // balance0Adjusted / 1000= balance0  - 3 * amount0In / 1000;
        // balance0Adjusted = balance0 * 1000 - 3 * amount0In
        uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
        uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
        // (x +  dx * (1-f)) * (y - dy) = x * y
        // balance0Adjusted / 1000 * balance1Adjusted / 1000 = reserve0 * reserve1;
        // balance0Adjusted * balance1Adjusted >= reserve0 * reserve1 * (1000 ** 2)
        require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }

```

### Swap Exercise 1:

Testing Swap function: 

```javascript
   
   function test_swapExactTokensForTokens() public {
      address[] memory path = new address[](3);

      address user = makeAddr("USER");

      path[0] = WETH;
      path[1] = DAI;
      path[2] = MKR;

      uint amountIn = 1e18;
      uint amountOutMin = 1;

      vm.prank(user);
      uint[] memory amounts = router.swapExactTokensForTokens({
         amountIn: amountIn,
         amountOutMin: amountOutMin,
         path: path,
         to: user,
         deadline: block.timestamp
      });


      console2.log("WETH",amounts[0]);
      console2.log("DAI",amounts[1]);
      console2.log("MKR",amounts[2]);

      assertGe(mkr.balanceOf(user),amountOutMin,"MKR BALANCE");

   }

```

### Swap Exercise 2:

Testing Swap function:


```javascript

   function test_swapTokensForExactTokens() public {
      address[] memory path = new address[](3);

      uint amountOut = 0.1 * 1e18 ;

      uint amountInMax = 1e18;

      address user = makeAddr("USER");

      path[0] = WETH;
      path[1] = DAI;
      path[2] = MKR;

      vm.prank(user);

      uint[] memory amounts = router.swapTokensForExactTokens({
         amountOut: amountOut,
         amountInMax: amountInMax,
         path: path,
         to: user,
         deadline: block.timestamp
      });

      console2.log("WETH",amounts[0]);
      console2.log("DAI",amounts[1]);
      console2.log("MKR",amounts[2]);

      assertGe(mkr.balanceOf(user),amountOut,"MKR BALANCE");
   }

```

## Spot Price Math:

### Spot Price(Mid price)

Spot Price = current price:
   - y/x = spot the price of token X in the term of token Y

   - execution price = tokens received / sent = -dy / dx --> Slope of green line

   - dy = dx * y / x + dx
   -  -dy / dx = - y / x + dx --> -y/x = slop of orange Line 
   -  when dx --> 0
   -  dy / dx = y/x for Small dx

## Slippage
- Difference Between the price you except to receive vs what you actually received

- Example :
   - Before Swap:
      amount in = 2000 DAI
      calculated amount out = 1 ETH

      - price of ETH = 2000 DAI / 1 ETH = 2000

   - After Swap:
      amount in = 2000 DAI
      calculated amount out = 0.99 ETH
       
      - price of ETH = 2000 DAI / 0.99 ETH = 2020

### Cause of Slippage:
- Market Movement: It Depend on transaction happening in the market.This is calculated By AMM(Automated Market Maker).
- It is easy to understood in the graph.