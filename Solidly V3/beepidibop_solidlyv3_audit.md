
Progress

- fee-changes
  - [x] Libraries
    - [x] Position
    - [x] Tick
  - [x] Factory
  - [x] Pool
  - [x] PoolDeployer

# Medium-risk Findings

## [M-1] `UniswapV3Pool`: `swap` does not revert when insufficient tokens are transferred to the pool

Transfer tax token swaps will transfer less tokens than required into the pool and still be accepted. This was already expected/discussed since Uniswap V3 explicitly states they don't support those tokens. But the swap would still revert since they do a balance check.

The lack of balance check would mean the swap would be successful even if the required amount doesn't reach the pool. There are also some ERC20s that don't revert on unsuccesful transfers, the pool currently would not revert on those swaps but give out free tokens. (e.g. [ZRX](https://etherscan.io/address/0xe41d2489571d322189246dafa5ebde1f4699f498#code#L64))

### Link To Affected Code

https://github.com/SolidlyV3/v3-core/blob/fee-changes/contracts/UniswapV3Pool.sol#L526

---

# Gas Findings

## [Gas-1] `Position`: `update()` can have some gas savings

Line 53-56 of `Position.sol` can be moved into the `else` block just a few lines above.
Line 46 isn't needed anymore since `liquidityNext` isn't used in any code afterwards

You can also remove defining `liquidityNext` on line 43 and just have the else block mutate it directly.

![](https://i.imgur.com/zdst7nv.png)

### Link To Affected Code

https://github.com/SolidlyV3/v3-core/blob/d06fa822f427895deabd5bd08a4f60ec5db620b9/contracts/libraries/Position.sol#L40

## [Gas-2] `Tick`: `cross()` doesn't seem to be needed anymore

Since all `cross()` does now is returning the tick's `liquidityNext` without writing to storage. You probably don't need to call the Tick library for this anymore, a `ticks[step.tickNext].liquidityNext` can replace `ticks.cross(step.tickNext)` in UniswapV3Pool.sol Line 473.

### Link To Affected Code

https://github.com/SolidlyV3/v3-core/blob/fee-changes/contracts/UniswapV3Pool.sol#L473

https://github.com/SolidlyV3/v3-core/blob/d06fa822f427895deabd5bd08a4f60ec5db620b9/contracts/libraries/Tick.sol#L86

## [Gas-3] `Pool`: `blockTimestamp` isn't needed anymore

They appear in `SwapCache` and as an internal function in the links below

You can also deprecate `SwapCache` as a struct if you remove `blockTimestamp`.

### Link To Affected Code

https://github.com/SolidlyV3/v3-core/blob/fee-changes/contracts/UniswapV3Pool.sol#L348

https://github.com/SolidlyV3/v3-core/blob/fee-changes/contracts/UniswapV3Pool.sol#L405

https://github.com/SolidlyV3/v3-core/blob/fee-changes/contracts/UniswapV3Pool.sol#L111

---

# Comments

## [Comment-1] Seems like some tests haven't been updated yet

**Tick.spec.ts:**

Looks like it's due to removing fee accounting and the test is still trying to verify fees.

**Oracle.spec.ts:**

Obsolete, but still passes because it's using the test contracts.

**UniswapV3Factory.spec.ts:**

Succeeds if you comment out line 85 calling pool.fee. Otherwise it's just gas and bytecode size difference, which is can just be updated in the snapshot.

**UniswapV3Pool.arbitrage.spec.ts:**

This isn't too serious, since I didn't see the swapping math changed. The results should be the same as if the tests were passing.

But to get the test to run, you'll need to rewrite how the testing contract supplies the tokens. The testing contract uses callbacks to perform swaps before but callbacks were removed due to code change.

**UniswapV3Pool.gas.spec.ts:**

Just need to update the snapshot to make this pass. Most gas snapshots are 10-30% lower than Uniswap V3, nice gas savings.

**UniswapV3Pool.spec.ts:**

Looks like more observation and fee related tests causing this to fail. Same comment as the arbitrage test, results should be the same as if it's passing.

**UniswapV3Pool.swaps.spec.ts:**

ditto.

**UniswapV3Router.spec.ts:**

Obsolete

## [Comment-2] Consider making pools by using tick spacing as input instead of fee

Just a nitpick, gets a bit confusing the in future when pools' fees change or when you enable new tick-spacing tiers
