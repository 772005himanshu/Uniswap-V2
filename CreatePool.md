# Create Pool

## Create Pair code:

```javascript

   function createPair(address tokenA, address tokenB) external returns (address pair) {
        require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
        // For checking both token are not same
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        // arrange them ascending order
        require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
        require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // checking they not pair already.If yes than another function should be called
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
        // CreationCode --> runtime code + constructor args
        // deploy with create2 - UniswapV2Library.pairFor
        bytes32 salt = keccak256(abi.encodePacked(token0, token1)); // it used create new contract address deterministically
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }// low level programming to make code more fast and gas efficient
        IUniswapV2Pair(pair).initialize(token0, token1); // if put in constructor args the while code point to the address of args. so we initialize with the help of function .
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // populate mapping in the reverse direction
        allPairs.push(pair);
        emit PairCreated(token0, token1, pair, allPairs.length);
    }

```

### Exercise Create Pair:

Testing Create Pair contract:

```javascript
   
    function test_createPair() public {
        ERC20 token = new ERC20("test","Test",18);

        // Write your code here
        ERC20 WETH = new ERC20("weth","WETH",0);

        address pair = factory.createPair({tokenA : address(token),
            tokenB: address(WETH)
        })


        address token0 = IUniswapV2Pair(pair).token0();
        address token1 = IUniswapV2Pair(pair).token1();

        if(address(token) < WETH){
            assertEq(token0,address(token),"token 0");
            assertEq(token1,WETH,"token 1");
        }
        else{
            assertEq(token1,WETH,"token 0");
            assertEq(token0,address(token),"token 1");
        }
    }


```


## Mint Pool Shares:

### Equation for Pool share after any liquidity provider give liquidity to the mempool How many share he get
- Formula,s = (L1 - L0 / L0) * T

### Math of Formula
T = Total shares,
s = shares to mint,
L0 = Value of Pool before,
L1 = Value of Pool after

- Mint `share` proportional to increase of L0 to L1 
    T + s / T = L1/L0 , assume T > 0
    - Solve for s :
        T + s / T = L1 / L0 -->
        s = L1 / L0 * T - T -->
        `s = (L1 - L0 / L0) * T`




## Burn Pool Share
1. How many USDC received? --> s/T * L0

### Math for Burn Pool Share
- Decrease L0 to L1 proportional to decrease of shares
- T - s / T = L1/L0
- after simplifying , `L0 - L1 = s/T * L0` by this we can calculate the USDC we get after get back there shares.


## Add Liquidity:

How many `dx` and `dy` to add?
    - `dy / dx = y / x`
    - if we draw a straight line from origin to current amount of AMM passes through the new amount of token x and new amount token y
    - price after add liquidity = price before
    - `y + dy / x + dx = y / x` after solving this equation you get `dy/dx = y/x`

### Pool Share

T = Total shares,
s = shares to mint,
L0 = Value of Pool before,
L1 = Value of Pool after
- `s = (L1 - L0 / L0) * T`
- `L1 - L0 / L0 = dx/x = dy/y`
- By upper Two Equation we get ,`s = dx/x * T = dy/y * T`
- 3 Way :
    - Define a function to measure pool value f(x,y) -> L
        - f(x,y) = sqrt(x*y)
        - f(x,y) = 2 * x
        - f(x,y) = 2 *y
    - f(x,y) = sqrt(x*y)
        - Motivation:
            - x * y = L ^ 2 --> sqrt(x*y) = L
            - L1 - L0 / L0 = f(x+dx,y+dy) - f(x,y) / f(x,y)
                           => sqrt((x+dx) * (y+dy)) - sqrt(x*y)  by dy = y/x * dx
                           we get, `L1 - L0 / L0 = dx/x` so, `s/T = dx/x`
    - f(x,y) = 2 * x = value of pool in token x
        - Motivation
              - x = 6,000,000 DAI
              - y = 3000 ETH
                - f(x,y) = x + (x/y)*y = 2 * x  , x/y = spot price of y in terms of x, place the value y in term of x
                        = 6,000,000 DAI + 6,000,000 DAI/ 3000 ETH * 3000 ETH
                        = 12,000,000 DAI
                - L1 - L0 / L0 = f(x+dx,y+dy) -f(x,y) / f(x,y)
                               = `2 * (x+dx) - 2 * x / 2 * x = dx`

## Add Liquidity
1. `Liquidity provider` call the function `addLiquidity` and call router contract
2. Router contract call factory Contract by the function `setPair and createPair`.if pair already exists then it pass to another step if not it call `create2` make their pair
3. Then `Liquidity provider` call `transferFrom` then `mint` function is called.
4. Last step should if `Liquidity provider` back their liquidity . Lp get reward of providing the liquidity to the pool

## Code for Add Liquidity:


```javascript
    
    // **** ADD LIQUIDITY ****
    function _addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin
    ) internal virtual returns (uint amountA, uint amountB) {
        // Check for pair of tokenA,tokenB if exist go on , if not createPair
        if (IUniswapV2Factory(factory).getPair(tokenA, tokenB) == address(0)) {
            IUniswapV2Factory(factory).createPair(tokenA, tokenB);
        }
        // getting reserve amount by call getReserves function
        (uint reserveA, uint reserveB) = UniswapV2Library.getReserves(factory, tokenA, tokenB);
        // if reserve amount == 0 then amountA = amountADesired , amountB = amountBDesired
        if (reserveA == 0 && reserveB == 0) {
            (amountA, amountB) = (amountADesired, amountBDesired);
        // NOTE: 
        // what is quote function is doing in the protocol
        // amountB = amountA.mul(reserveB) / reserveA; = dy = (y / x) * dx
        } else {
            uint amountBOptimal = UniswapV2Library.quote(amountADesired, reserveA, reserveB);
            // amountBMin <=amountBOptimal <= amountBDesired
            if (amountBOptimal <= amountBDesired) {
                require(amountBOptimal >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
                (amountA, amountB) = (amountADesired, amountBOptimal);
            } else {
                uint amountAOptimal = UniswapV2Library.quote(amountBDesired, reserveB, reserveA);
                assert(amountAOptimal <= amountADesired);
                require(amountAOptimal >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
                (amountA, amountB) = (amountAOptimal, amountBDesired);
            }
        }
    }
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
        (amountA, amountB) = _addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin); // see upper explain
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB); // getting pair address 
        TransferHelper.safeTransferFrom(tokenA, msg.sender, pair, amountA); // we first transfer toke to pair contract then we go to mint
        TransferHelper.safeTransferFrom(tokenB, msg.sender, pair, amountB);
        liquidity = IUniswapV2Pair(pair).mint(to);
    }


    // this low-level function should be called from a contract which performs important safety checks
   function mint(address to) external lock returns (uint liquidity) {
        // gas savings
        (uint112 reserve0, uint112 reserve1,) = getReserves();
        // gas savings
        uint balance0 = IERC20(token0).balanceOf(address(this));
        // gas savings
        uint balance1 = IERC20(token1).balanceOf(address(this));
        // NOTE: amounts are calculated by taking differences from internal balances
        uint amount0 = balance0.sub(reserve0);
        uint amount1 = balance1.sub(reserve1);

        bool feeOn = _mintFee(reserve0, reserve1);
        // gas savings, must be defined here since totalSupply can update in _mintFee
        uint totalSupply = totalSupply; 
        if (totalSupply == 0) {
          // NOTE: pool value function f(x, y) = sqrt(x * y)
          // NOTE: Math.sqrt(amount0 * amount1) = sub MINIMUM_LIQUIDITY
          // NOTE: protection against vault inflation attack - what is meant by vault inflation attack -> attacker can manipulate the vault for upcoming user
          // so we stored the minimum liquidity in address(0)
          liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
          _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
        } else {
          // NOTE: Math.min(amount0 * totalSupply / reserve0, amount1 * totalSupply / reserve1)
          liquidity = Math.min(amount0.mul(totalSupply) / reserve0, amount1.mul(totalSupply) / reserve1);
        }
        require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
        _mint(to, liquidity);
        update(balance0, balance1, reserve0, reserve1);
        // NOTE: used for protocol fee xy=k'L^2/xy or k'=K*(1+fee)/ (1+fee)^2
        if (feeOn) kLast = uint(reserve0).mul(reserve1) / totalSupply; // reserve0 and reserve1 are up-to-date
        // NOTE: msg.sender and amount0, amount1 reserve
        emit Mint(msg.sender, amount0, amount1);
    }


```


### Testing for Add Liquidity

```javascript

    function test_addLiquidity() public {

        ERC20 DAI = new ERC20("dai","DAI",18);
        ERC20 WETH = new ERC20("weth","WETH",18);

        uint constant DESIRED_A_AMOUNT = 1e6 * 1e18;
        uint constant DESIRED_B_AMOUNT = 100 * 1e18;

        vm.prank(user);

        (uint amountA, uint amountB, uint liquidity) = router.addLiquidity({tokenA: address(DAI),tokenB: address(WETH) ,amountADesired: DESIRED_A_AMOUNT  ,amountBDesired:DESIRED_B_AMOUNT,amountAMin: 1,amountBMin : 1 ,to: user,deadline: block.timestamp});

        console2.log("DAI",amountA);
        console2.log("WETH",amountB);
        console2.log("LP",liquidity );

        assertGt(pair.balanceOf(user),0,"LP = 0")
    }


```


