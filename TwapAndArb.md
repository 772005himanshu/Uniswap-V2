## Twap And Arb

### Danger of using Uniswap V2 spot price for pride oracle
- Danger = easy to manipulate spot price
- max borrow = 80 % of collateral value in DAI spot price
- Attacker can manipulate it by giving high amount of same token and make the price of another token very high use this as a profit

### Uniswap V2 TWAP (time weighted average price)
- Pi = Price of token X in terms of token Y between time t(i)<= t < t(i+1)
- Δ of T of i = T of i + 1 - T of i
- If we want to calculate TWAP from t0 to tn we take th average of P0,P1,P2 and so on.`P0 + P1 + P2 + ... + P to the n - 1`
- we have to multiply each term with `Δ of T of i / T of n - T of 0` and here i change with term index.
- The final form should be:
    = `Σ( i = k to n - 1 ) Δ of Ti * Pi / Tn - Tk`


#### Cumulative Price
- Cj = cumulative price up to ` tj = Σ( i = k to n - 1 ) Δ of Ti * Pi`
    - `Σ( i = k to n - 1 ) Δ of Ti * Pi` = Simple math we write it as , `C(n) - C(k)`
    - So formula should be = `C(n) - C(k) / Tn - Tk`

### TWAP to current time,t
- set t = tn+1 and current price P = Pn
- TWAP from Tk to Tk+1 = Cn+1 - Ck/Tn+1 - Tk
    - Cn+1 = `Σ( i = k to n ) Δ of Ti * Pi` = `Σ( i = k to n - 1 ) Δ of Ti * Pi + Δ of Tn * Pn` = `Cn + (T - Tn)P`
    - `Twap from Tk to Tk+1 = Cn - (T - Tn) * P - Ck / T - Tk`
    - `1/Twap of X != Twap of X`
  
#### TWAP contract(exercise):

```javascript
    
    contract uniswapV2Twap {

        using FixedPoint for *; // for using UQ112 * UQ112 contract to take decimal digit 

        uint256 private constant MIN_WAIT = 300;

        IUniswapV2Pair public immutable pair;

        address public immutable token0;
        address public immutable token1;

        // cumulative prices are uq112xuq112 price * seconds
        uint256 public price0CumulativeLast;
        uint256 public price1CumulativeLast;

        uint32 public updateAt;

        FixedPoint.uq112x112 public price0Avg; // uq112 is simply a contract for taking decimals for a number uq112*112 == 2**112 ,means sifting to the left 

        FixedPoint.uq112x112 public price1Avg;

        // Exercise 1
        constructor(address _pair) {
            pair = IUniswapV2Pair(_pair);

            token0 = pair.token0();
            token1 = pair.token1();

            price0CumulativeLast = pair.price0CumulativeLast();

            price1CumulativeLast = pair.price1CumulativeLast();
            (,,updatedAt) = pair.getReserves();


        }

        // Exercise 2
        function _getCurrentCumulativePrices() internal view returns(uint256 price0Cumulative, uint256 price1Cumulative){

            price0Cumulative = pair.price0CumulativeLast();
            price1Cumulative = pair.price1CumulativeLast();

            (uint112 reserve0,uint112 reserve1, uint32 blockTimestampLast) = pair.getReserves();

            // Cast Block.timestamp to uint32
            uint32 blockTimestamp = uint32(block.timestamp);

            if(blockTimestampLast != blockTimestamp) {
                // calculate elapsed time
                uint32 dt = blockTimestamp - blockTimestampLast;


                unchecked{

                    price0Cumulative += uint256(FixedPoint.fraction(reserve1, reserve0)._x) * dt;
                    price1Cumulative += uint256(FixedPoint.fraction(reserve0, reserve1)._x) * dt;

                }

            }
        }

        // Exercise 3
        function update() public{
            uint32 blockTimestamp = uint32(block.timestamp);
            // calculate elapsed time since last time cumulative prices were updated to this contract
            uint32 dt = blockTimestamp - updatedAt;

            require(dt >= MIN_WAIT,"time collapsed");

            (uint256 price0Cumulative , uint256 price1Cumulative) = _getCurrentCumulativePrices();


            unchecked {
                price0Avg = FixedPoint.uq112x112(uint224((price0Cumulative - price0CumulativeLast)/dt));
                price1Avg = FixedPoint.uq112x112(uint224((price1Cumulative - price1CumulativeLast)/dt));
            }

            // update state variables price0CumulativeLast,price1CumulativeLast,updatedAt
            price0CumulativeALst = price0Cumulative;
            price1CumulativeALst = price1Cumulative;
            updatedAt = blockTimestamp;

        }


        // Exercise 4

        function consult(address tokenIn, uint256 amountIn) external view returns(uint256 amountOut) {
            // Require token either token0 or token1
            require(token == token0 || token == token1, "invalid token");

            // calculate amountOut
            // mul is function containing only only struct of uq144x112 for converting in 144 we use decode function 
            if(token == token0){
                amountOut = FixedPoint.mul(price0Avg,amountIn).decode144();
            }else{
                amountOut = FixedPoint.mul(price1Avg,amountIn).decode144();
            }

        }

    }

```

## Arbitrage
- Taking profit by small amount of change in two contract price of token

``` 
    graph LR
    A[Uniswap V2] --> B[User]
    B[User] --> C[SushiSwap]
    A[Uniswap V2] -- 3000 DAI / WETH --> B[User]
    B[User] -- 3100 DAI / WETH --> C[SushiSwap]

```

- We can also use flash loan/swap for buying assets

```javascript

    contract uniswapV2Arb1 {
        struct SwapParams {
            // Router to execute first swap - tokenIn for tokenOut
            address router0;
            // Router to execute second swap - tokenOut for tokenOut
            address router1;
            // token in first swap
            address tokenIn;
            // token swap in second swap
            address tokenOut;
            // amount in for the first swap
            uint256 amountIn;
            // revert the arbitrage if profit is less than this minimum
            uint256 minProfit;
        }
        // internal function for sing in other function
        function _swap(SwapParams memory params) private returns(uint256 amountOut) {
            // Swap on router0 (tokenIn -> tokenOut)
            IERC20(params.tokenIn).approve(params.router0.params.amountIn);
            address[] memory path = new address[](2);
            path[0] = params.tokenIn;// input of every swap
            path[1] = params.tokenOut;// output of every swap
            uint256[] memory amounts = IUniswapV2Router02(params.router0).swapExactTokensForTokens{(
                amountIn: params.amountIn,
                amountOutMin: 0,
                path: path,
                to: address(this),
                deadline: block.timestamp
            )};
            // swap on router1 (tokenOut -> tokenIn)
            IERC20(params.tokenOut).approve(params.router1,amounts[1]);
            path[0] = params.tokenOut;// it input of second swap and output of first swap
            path[1] = params.tokenIn; // it should be output from second swap
            amounts = IUniswapV2Router02(params.router1).swapExactTokensForTokens{(
                amountIn: amounts[1],
                amountOutMin: params.amountIn,
                path: path,
                to: address(this),
                deadline: block.timestamp
            )};

            amountOut = amounts[1];
        }
        // Exercise 1
        function swap(SwapParams calldata params) external {
            address user = makeAddr("USER");

            IERC20(params.tokenIn).transferFrom(msg.sender,address(this),params.amountIn);

            uint256 amountOut = _swap(params);
            require(amountOut - params.amountIn >= params.minProfit."minProfit > profit");
            IERC20(params.tokenIn).transfer(msg.sender,amountOut);


        }


        // exercise 2
        function flashSwap(address pair,bool isToken0, SwapParams calldata params) external {
            bytes memory data = abi.encode(msg.sender,pair.params)

            IUniswapV2Pair(pair).swap({
                amount0Out : isToken0 ? params.amountIn : 0 ,
                amount1Out : isToken0 ? 0 : params.amountIn ,
                to : address(this),
                data : data

            })
        }

        function uniswapV2Call(address sender, uint256 amount0Out,uint256 amount1Out,bytes memory data) external {
            // calling the contract for swap and other things
            (address caller , address pair, SwapParams memory params) = abi.decode(data , (address , address , SwapParams));

            uint256 amountOut = _swap(params);

            uint256 fee = ( (params.amountIn * 3) / 997 ) + 1;
            uint256 amountToRepay = params.amountIn + fee ;
            
            uint256 profit = amountOut - amountToRepay
            require(profit >= params.minProfit, "profit < min");

            IERC20(params.tokenIn).transfer(address(pair),amountToRepay);
            IERC20(params.tokenIn).transfer(caller,profit);

        }

    }


```

- Arb Ex -2 on implementing a Uniswap V2 arbitrage on Arbitrage

```javascript
    contract uniswapV2ArbV2 {
        struct FlashSwapData {
            // Caller of flashSwap (msg.sender inside flashSwap)
            address caller;
            // Pair to flash swap from
            address pair0
            // Pair to swap from
            address pair1
            // True if flash swap is token0 in and token1 out
            bool isZeroForOne;
            // Amount in to repay flash swap
            uint256 amountIn;
            // Amount to borrow from flash swap
            uint256 amountOut;
            // Revert if Profit is less than this minimum
            uint256 minProfit
        }

        function flashSwap(
            address pair0,
            bool isZeroForOne,
            uint256 amountIn,
            uint256 minProfit
        ) external {
            // Write your code here
            // Don't change any other code
            (uint112 reserve0,uint112 reserve1,) = IUniswapV2Pair(pair0).getReserves();
            // Hint - use getAmountOut to calculate amountOut to borrow

            uint256 amountOut = isZeroForOne ? getAmountOut(amountIn,reserve0,reserve1):(amountIn,reserve1,reserve0);

            bytes memory data = abi.encode(FlashSwapData({
                    caller : msg.sender,
                    pair0: pair0,
                    pair1: pair1 ,
                    isZeroForOne: isZeroForOne,
                    amountIn : amountIn,
                    amountOut : amountOut,
                    minProfit : minProfit
                })
            );


            IUniswapV2Pair(pair0).swap({
                amount0Out : isZeroForOne ? 0 : amountOut,
                amount1Out : isZeroForOne ? amountOut : 0 ,
                to: address(this),
                data: data
            })
        }

        function uniswapV2Call(
            address sender,
            uint256 amount0Out,
            uint256 amount1Out,
            bytes calldata data
        ) external {
            // Write your code here
            // Don't change any other code
            FlashSwapData memory params = abi.decode(data,(FlashSwapData));

            address token0 = IUniswapV2Pair(params.pair0).token0();
            address token1 = IUniswapV2Pair(params.pair0)token1();

            (address tokenIn, address tokenOut) = params.isZeroForOne ? (token0,token1) : (token1,token0);

            IERC20(tokenOut).transfer(params.pair1,params.amountOut);

            UniswapV2Pair(params.pair1).swap(
                amountOut,
                amount0Out,
                amount1Out,
                address(this),
                data
            );

            IERC20(tokenIn).transfer(params.pair0,params.amountIn);

            uint256 profit = amountOut - params.amountIn;

            require(profit >= params.minProfit, "profit < min");

            IERC20(tokenIn).transfer(params.caller.profit);

        }

        function getAmountOut(
            uint256 amountIn,
            uint256 reserveIn,
            uint256 reserveOut
        ) internal pure returns (uint256 amountOut) {
            uint256 amountWithFee = amountIn * 997;
            uint256 numerator = amountWithFee * reserveOut;
            uint256 denominator = reserveIn * 1000 + amountWithFee;
            amountOut = numerator / denominator;
        }
    }


```

#### Arbitrage Optimal AmountIn :
 
```javascript
    dy_a* = (-b + sqrt(b^2 - 4ac)) / 2a 
```

- a = k ^ 2
- b = 2 * k * x_a *y _b
- c =(y_a * x _b) + ((1-f) * (x_b) + (1 - f) * x_a) ^ 2