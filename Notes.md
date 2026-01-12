# [Basic]

### [B-1] Etherscan::Transaction

**Description** 对于ETH来说，区分交易的类别是很重要的。

#### [Transaction type classification]

- **🛜Type0 [Legacy Transaction]**

  最早的以太坊交易形式，使用 **单一 Gas Price**，没有 Base Fee / Priority Fee 的概念，手续费 = `Gas Used × Gas Price`，现在仍然**兼容**，但不推荐使用。这也是很多链下服务出bug的一个点，协议太老了，不适配新协议。下面的表格着重看gasPrice和gasLimit

  | 字段     | 说明          |
  | -------- | ------------- |
  | gasPrice | 固定 gas 单价 |
  | gasLimit | Gas 上限      |
  | nonce    | 交易序号      |
  | to       | 接收地址      |
  | value    | ETH 数量      |
  | data     | 合约数据      |

- 🛜 **Type1 [Access List Transaction (EIP-2930)]**

  核心概念是AccessList，这个会直接声明要访问的数据

  ```yaml
  [
    {
      address: 0xContractA,
      storageKeys: [slot1, slot2, ...]
    },
    {
      address: 0xContractB,
      storageKeys: [...]
    }
  ]
  
  ```

  

- 🛜 **Type 2：EIP-1559 Transaction (主流)** 

  引入 **Base Fee（销毁）**，引入 **Priority Fee（矿工小费）**，自动退还多余 Gas

  ```js
  effectiveGasPrice =
  min(
    maxFeePerGas
    baseFee + maxPriorityFeePerGas
  )
  ```

  | 字段                 | 含义                 |
  | -------------------- | -------------------- |
  | maxFeePerGas         | 你愿意支付的最高 Gas |
  | maxPriorityFeePerGas | 给矿工的小费         |
  | baseFee              | 网络自动决定         |

  相当于原来的`gasPrice`被拆分成了`maxFeePerGas`和`maxPrioityFeePerGas`，实际的gas Fee

- 🛜 **Type 3：Blob Transaction（EIP-4844 / Proto-Danksharding）**

  2024年引入，面向layer2，数据放在blob中，隔一段时间主网会删除Blob

  Rollup（如 Arbitrum、Optimism）提交数据，数据放在 **Blob** 中，而不是 calldata，极低的数据成本，不直接参与 EVM 执行，专为扩容设计。给 Rollup（如 Arbitrum、Optimism）提交数据，数据放在 **Blob** 中，而不是 calldata。极低的数据成本，不直接参与 EVM 执行，专为扩容设计

#### 





# [PANPTIC]

### [SK-PANOPTIC-1] Use BytesMask for more efficient storage

**Description:** `BytesMasking` is a technique to pack mulitple values into a single storage slot(usually taking uint256 -> 32 bytes == 256 bits) to save gas. instead of using seperate storage slots for each variable (such us struct)

<details>
<summary>💹Example in Panoptic</summary>

```js
PACKING RULES FOR A MARKETSTATE:
From the LSB to the MSB:
(0) borrowIndex          80 bits : Global borrow index in WAD (starts at 1e18). 2**80 = 1.75 years at 800% interest
(1) marketEpoch          32 bits : Last interaction epoch for that market (1 epoch = block.timestamp/4)
(2) rateAtTarget         38 bits : The rateAtTarget value in WAD (2**38 = 800% interest rate)
(3) unrealizedkkkkkInterest   106bits : Accumulated unrealized interest that hasn't been distributed (max deposit is 2**104)
Total                    256bits  : Total bits used by a MarketState.
```
The `MarketState` packs 4 values into 1 storage slot:

|<--0-79-->|<--80-111-->|<--112-149-->|<--150-155-->|

| 80 Bits  |  32 Bits   |  32 Bits    |   106 Bits  |

</details>

<details>
<summary>💹How to use this powerful skill?</summary>

Example:
The `MarketState` packs 4 values into 1 storage slot:

|<--0-79-->|<--80-111-->|<--112-149-->|<--150-155-->|

| 80 Bits  |  32 Bits   |  38 Bits    |   106 Bits  |

First, define the mask we need. In this case, rateAtTarget is required 

```js
TARGET_RATE_MASK = ((1 << 38) - 1) << 112;
// creates: 111...111(38 ones) at position 112-149
// 0x...3FFFFFFFFF000000000000000000000000000
```

Then, we use Yul to load sepecific storage 

- Write Value by mask

```js
MarketState self,
uint40 newRate

assembly{
    //clear bits 112-149
    let cleared := and(selt, not(TARGET_RATE_MASK));
    // ...000..000(38 zero in position)

    //2. Mask the input to ensure it fits 38 bits
    // uint40 -> we have to ignore the top 2 bits
    // 0011 1111 1111 ....  1111 1111
    let safeRate := and(newRate, 0x3FFFFFFFFF);


    let result := or(cleared,shl(112, safeRate));
}

```

- Read Value by mask

```js
MarketState selt

assembly{
    // push->[xxx ...|         RATE          |] 
    //               0011 1111 .... 1111 1111
    //               &&&& &&&& &&&& &&&& &&&&
    let result := and(shr(112,self), 0x3FFFFFFF); 
}

```
</details>

<br>

**Benefits:** 

1. Gas Savings: SSTORE (~20k gas)  vs $ SSTORE
2. Atomic Update: All values update togeter
3. Cache Efficiency: Reading mulitple values cost less


# [Uniswap Introduction]

## Uniswap V2

# Uniswap Introduction

## Uniswap V2

### [UNIV2-1] 为什么需要两个 codebase？

**Discription:** uniswap V2 有两个仓库，`v2-core`和`v2-periphery`。区分二者的重点在于面向对象的不同。v2-core 是核心，里面包含了 pool 的创建，token swap 逻辑，其中的 function 普通用户是用不上的。v2-periphery 专门用用来与用户交互。

### [UNIV2-2] Swap Token in V2

**Discription:** `v2-periphery`提供接口给用户 swapTokens，分别是`UniswapV2RouterV2::swapExactTokensForTokens()`以及`UniswapV2RouterV2::swapTokensForExactTokens()` .

- swapExactTokenForToken() 目的是通过 exactInput -> 计算出 calculated output，然后交易。 举个例子，我有 1WETH，我要用这 1WETH 去兑换 DAI，在这个场景下我不知道我能兑换多少 DAI，但是我会提供指定的 WETH。由于兑换出的 DAI 是未知数，uniswapv2 提供了滑点保护(slip protection)用于抵抗 MEV,说人话就是我(用户)可以指定一个最低兑换 DAI 的数量，如果小于这个数就放弃。
- swapTokenForExactToken() 目的是通过 exactOutput -> 计算出 calculated input，然后交易。

<details>
<summary>💹Swap?TokenFor?Token</summary>

```js
function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint[] memory amounts){
        ...

        _swap(amounts, path, to);
    }

function swapTokensForExactTokens(
        uint amountOut,
        uint amountInMax,
        address[] calldata path,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint[] memory amounts){
        ...

        _swap(amounts, path, to);
    }
```

**Inspect:** 通过上面两个 function 我们发现，交易并不是 1 对 1 对，而是 1->1->1..->1,我们可以传[WETH, USDC, DAI]。这样最终结果还是 WETH -> DAI。出现这种场景是因为没有 WETH/DAI 的 pool，所以只能“绕远路”。  
最后我们观察`_swap(...)`这个函数，入参的 path 和 to 都是有的，amounts 是哪里来的？
答案我省略了 hhh，这个 amounts 其实就是 path[i]->path[i+1]的钱(正向，也就是 exactTokenforToken) ||path[i]<-path[i+1]的钱(逆向，也就是 TokenforExactToken),这其中的钱 uniswap 会计算好。

</details>
</details>

<details>

<summary>💹_swap in Router</summary>

```js
function _swap(uint[] memory amounts, address[] memory path, address _to) internal virtual {
        for (uint i; i < path.length - 1; i++) {
            // index i ->path[i]::path[i+1]
            // input = path[i], output = path[i + 1]
            (address input, address output) = (path[i], path[i + 1]);
            // in Uniswap Pair, the smaller address of token will be regarded as token0
            (address token0, ) = UniswapV2Library.sortTokens(input, output);
            // token0 is the smaller address of the two tokens
            uint amountOut = amounts[i + 1];
            // if input is the smaller one, then inputOutcom e is 0; if input != the smaller one, thus the logic is token1 swap token0
            (uint amount0Out, uint amount1Out) = input == token0 ? (uint(0), amountOut) : (amountOut, uint(0));
            // next pair or receiver address
            address to = i < path.length - 2 ? UniswapV2Library.pairFor(factory, output, path[i + 2]) : _to;
            IUniswapV2Pair(UniswapV2Library.pairFor(factory, input, output)).swap(
                amount0Out,
                amount1Out,
                to,
                new bytes(0)
            );
        }
    }
```

**Inspect:** `UniswapV2RouterV2::_swap`这个循环调用 v2-core 的 swap 函数，这也是为什么说`periphery`面向用户，核心的状态改变都在`v22-core`中

</details>
</details>

<details>

<summary>💹swap in Pair</summary>

```js
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
        require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');

        //_reserve0 = X0, _reserve1 = Y0
        (uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
        //amount0Out = token0 output
        //amount1Out = token1 output
        require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

        uint balance0;
        uint balance1;
        //scope for _token{0,1}, avoids `stack too deep` errors
        {
            address _token0 = token0;
            address _token1 = token1;
            require(to != _token0 && to != _token1, 'UniswapV2: INVALID_TO');
            if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
            if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
            if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
            balance0 = IERC20(_token0).balanceOf(address(this));
            balance1 = IERC20(_token1).balanceOf(address(this));
        }
        //      In         after  >   before  -  transfer  ?  after   - (   before - transfer  ) : 0
        //`after > before - transfer` means the token0Pool received tokens so the input is not 0
        uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
        {
            // scope for reserve{0,1}Adjusted, avoids stack too deep errors
            uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
            uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
            require(
                // Invariant x*y=L^2
                balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000 ** 2),
                'UniswapV2: K'
            );
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
```

**Inspect:** 在一个 Pair 的合约实例中，两个 token 需要按照 address 的地址排序，排序成 token0 和 token1。入参的 amount0Out 和 amount1Out 有一个是 0，函数的逻辑会计算 amount1In 和 amount0In。在 swap 的过程中会收取 0.3%的手续费

</details>
</details>

由此,swap 的逻辑简单梳理了一遍

### [UNIV2-3] TWAP (time weight everage price) in UniswapV2

**Discription:** 在使用 Uniswap 这种链上 Oracle 最为 price 来源的时候，很容易(100%)会受到攻击，原因就在于 Uniswap 的价格太好操控了，任何一个人做 FlashLoan 就可以让价格波动很大。由此 Uniswap 提供`TWAP`(time weight everage price)来防止价格波动。注意，TWAP 价格和现货价格是两个东西。

**Math:**

- **Spot Price 现货价格(AMM)**

  - Token X 以 Token Y 计价的现货价格:
    $$
    P\_{X/Y} = \frac{Y}{X}
    $$

- **TWAP 价格**

  - Token X 在时间区间 i 到 k 的时间加权平均价格?
    $$
    \text{TWAP}_X(T_k, T_n) = \frac{\sum\limits_{i=k}^{n-1} \Delta T_i \, P_i}{T_n - T_k}
    $$

<details>
<summary>💹 _update in pair</summary>

```js
function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
        require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
        uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
        if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
            // * never overflows, and + overflow is desired
            price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
            price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
        }
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        blockTimestampLast = blockTimestamp;
        emit Sync(reserve0, reserve1);
    }
```

​	</details>

<details>
<summary>💹 How to use TWAP in your dapp</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity >=0.4 <0.9;

import {IUniswapV2Pair} from "../../../src/interfaces/uniswap-v2/IUniswapV2Pair.sol";
import {FixedPoint} from "../../../src/uniswap-v2/FixedPoint.sol";

// Modified from https://github.com/Uniswap/v2-periphery/blob/master/contracts/examples/ExampleOracleSimple.sol
// Do not use this contract in production
contract UniswapV2Twap {
    using FixedPoint for *;

    // Minimum wait time in seconds before the function update can be called again
    // TWAP of time > MIN_WAIT
    uint256 private constant MIN_WAIT = 300;

    IUniswapV2Pair public immutable pair;
    address public immutable token0;
    address public immutable token1;

    // Cumulative prices are uq112x112 price * seconds
    uint256 public price0CumulativeLast;
    uint256 public price1CumulativeLast;
    // Last timestamp the cumulative prices were updated
    uint32 public updatedAt;

    // TWAP of token0 and token1
    // range: [0, 2**112 - 1]
    // resolution: 1 / 2**112
    // TWAP of token0 in terms of token1
    FixedPoint.uq112x112 public price0Avg;
    // TWAP of token1 in terms of token0
    FixedPoint.uq112x112 public price1Avg;

    // Exercise 1
    constructor(address _pair) {
        // 1. Set pair contract from constructor input
        pair = IUniswapV2Pair(_pair);
        // 2. Set token0 and token1 from pair contract
        token0 = pair.token0();
        token1 = pair.token1();
        // 3. Store price0CumulativeLast and price1CumulativeLast from pair contract
        price0CumulativeLast = pair.price0CumulativeLast();
        price1CumulativeLast = pair.price1CumulativeLast();
        // 4. Call pair.getReserve to get last timestamp the reserves were updated
        (, , updatedAt) = pair.getReserves();
        //    and store it into the state variable updatedAt
    }

    // Exercise 2
    // Calculates cumulative prices up to current timestamp
    //@note 这个函数计算并返回截止到当前时间戳的累积价格，用于后续计算时间加权平均价格。
    function _getCurrentCumulativePrices()
        internal
        view
        returns (uint256 price0Cumulative, uint256 price1Cumulative)
    {
        // 1. Get latest cumulative prices from the pair contract
        price0Cumulative = pair.price0CumulativeLast();
        price1Cumulative = pair.price1CumulativeLast();
        // If current block timestamp > last timestamp reserves were updated,
        // calculate cumulative prices until current time.
        // Otherwise return latest cumulative prices retrieved from the pair contract.

        // 2. Get reserves and last timestamp the reserves were updated from
        //    the pair contract
        (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast) = pair
            .getReserves();

        // 3. Cast block.timestamp to uint32, and update the timestamp of the last update
        uint32 blockTimestamp = uint32(block.timestamp);
        if (blockTimestampLast != blockTimestamp) {
            // 4. Calculate elapsed time
            uint32 dt = blockTimestamp - blockTimestampLast;

            // Addition overflow is desired
            unchecked {
                // 5. Add spot price * elapsed time to cumulative prices.
                //    - Use FixedPoint.fraction to calculate spot price.
                //    - FixedPoint.fraction returns UQ112x112, so cast it into uint256.
                //    - Multiply spot price by time elapsed
                price0Cumulative +=
                    uint256(FixedPoint.fraction(reserve1, reserve0)._x) *
                    dt;
                price1Cumulative +=
                    uint256(FixedPoint.fraction(reserve0, reserve1)._x) *
                    dt;
            }
        }
    }

    // Exercise 3
    // Updates cumulative prices
    function update() external {
        // 1. Cast block.timestamp to uint32
        uint32 blockTimestamp = uint32(block.timestamp);
        // 2. Calculate elapsed time since last time cumulative prices were
        //    updated in this contract
        uint32 dt = blockTimestamp - updatedAt;
        // 3. Require time elapsed >= MIN_WAIT
        require(dt >= MIN_WAIT, "InsufficientTimeElapsed");

        // 4. Call the internal function _getCurrentCumulativePrices to get
        //    current cumulative prices
        (
            uint256 price0Cumulative,
            uint256 price1Cumulative
        ) = _getCurrentCumulativePrices();

        // Overflow is desired, casting never truncates
        // https://docs.uniswap.org/contracts/v2/guides/smart-contract-integration/building-an-oracle
        // Subtracting between two cumulative price values will result in
        // a number that fits within the range of uint256 as long as the
        // observations are made for periods of max 2^32 seconds, or ~136 years
        unchecked {
            // 5. Calculate TWAP price0Avg and price1Avg
            //    - TWAP = (current cumulative price - last cumulative price) / dt
            //    - Cast TWAP into uint224 and then into FixedPoint.uq112x112
            price0Avg = FixedPoint.uq112x112(
                uint224(price0Cumulative - price0CumulativeLast) / dt
            );
            price1Avg = FixedPoint.uq112x112(
                uint224(price1Cumulative - price1CumulativeLast) / dt
            );
        }

        // 6. Update state variables price0CumulativeLast, price1CumulativeLast and updatedAt
        price0CumulativeLast = price0Cumulative;
        price1CumulativeLast = price1Cumulative;
        updatedAt = blockTimestamp;
    }

    // Exercise 4
    // Returns the amount out corresponding to the amount in for a given token
    function consult(
        address tokenIn,
        uint256 amountIn
    ) external view returns (uint256 amountOut) {
        // 1. Require tokenIn is either token0 or token1
        require(tokenIn == token0 || tokenIn == token1, "InvalidToken");
        // 2. Calculate amountOut
        //    - amountOut = TWAP of tokenIn * amountIn
        //    - Use FixePoint.mul to multiply TWAP of tokenIn with amountIn
        //    - FixedPoint.mul returns uq144x112, use FixedPoint.decode144 to return uint144
        if (tokenIn == token0) {
            // Example
            //   token0 = WETH
            //   token1 = USDC
            //   price0Avg = avg price of WETH in terms of USDC = 2000 USDC / 1 WETH
            //   tokenIn = WETH
            //   amountIn = 2
            //   amountOut = price0Avg * amountIn = 4000 USDC
            amountOut = FixedPoint.mul(price0Avg, amountIn).decode144();
        } else {
            amountOut = FixedPoint.mul(price1Avg, amountIn).decode144();
        }
    }
}
```

</details>

## Uniswap V3







