# [PANPTIC]

## [SK-PANOPTIC-1] Use BytesMask for more efficient storage

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
