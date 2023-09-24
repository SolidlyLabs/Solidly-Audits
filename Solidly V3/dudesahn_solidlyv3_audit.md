# Solidly UniV3 Update Audit
## Findings
### Lack of balance check in `swap` function [M]
- **Issue:** In several places, `safeTransfer` is replaced by a simple `transfer` function, presumably as a gas optimization. Additionally, UniswapV3's callback (which is required to transfer tokens) is removed, and replaced with a simple `transferFrom` of the token transfer previously handled in the fallback. As a part of removing the fallback, the previous checks to assert that balances were correctly handled has also been removed. The combined lack of balance checks and no use of `safeTransfer` means that there is no longer a guarantee that the pool receives the funds it should for non-standard ERC20s.
    - Previous check: `if (balance1Before + uint256(amount1) > balance1())`
- **Suggested Fix:** Use `safeTransferFrom` when transferring tokens from msg.sender to the pool.
- **Code to Check:** Lines [525-531](https://github.com/SolidlyV3/v3-core/blob/29c9c3c03c6d6849da7f3e13b488e3175298c9a7/contracts/UniswapV3Pool.sol#L525-L531)


### No use of `safeTransfer` [L]
- **Issue:** See above, this check is removed elsewhere than just lines 526 and 531.
- **Suggested Fix:** Use `safeTransfer`.
    - Please note that several tokens do not conform to the standard ERC20 interface, and thus it is recommended to use safeTranser to accomodate these edge cases. For more about the reasons for using safeTransfer, see a discussion on OpenZeppelin's forums [here](https://forum.openzeppelin.com/t/making-sure-i-understand-how-safeerc20-works/2940).
- **Code to Check:** Line [302](https://github.com/SolidlyV3/v3-core/blob/29c9c3c03c6d6849da7f3e13b488e3175298c9a7/contracts/UniswapV3Pool.sol#L302), Line [306](https://github.com/SolidlyV3/v3-core/blob/29c9c3c03c6d6849da7f3e13b488e3175298c9a7/contracts/UniswapV3Pool.sol#L306), Line [526](https://github.com/SolidlyV3/v3-core/blob/29c9c3c03c6d6849da7f3e13b488e3175298c9a7/contracts/UniswapV3Pool.sol#L526), Line [531](https://github.com/SolidlyV3/v3-core/blob/29c9c3c03c6d6849da7f3e13b488e3175298c9a7/contracts/UniswapV3Pool.sol#L531), Line [549](https://github.com/SolidlyV3/v3-core/blob/29c9c3c03c6d6849da7f3e13b488e3175298c9a7/contracts/UniswapV3Pool.sol#L549), Line [550](https://github.com/SolidlyV3/v3-core/blob/29c9c3c03c6d6849da7f3e13b488e3175298c9a7/contracts/UniswapV3Pool.sol#L550)


### Difficult to know all addresses currently trusted by factory/pools [L]
- **Issue:** The use of the newly added fee controls for any pool are limited to only to addresses approved by a given factory. The `onlyTrusted` modifier reads from the `isTrusted` mapping on the factory, and tells the pool whether a given address is allowed to alter and claim fees. While addresses can be flipped between true and false (0 and 1) in the `isTrusted` mapping, to securely audit which addresses are trusted at any given time (ie, to make sure old addresses were removed), one would need to use events or inspect changes to storage slots.
- **Suggested Fix:** So that all trusted addresses can be inspected on-chain, it is recommended to use an array to store trusted addresses.
    - The quick and dirty fix would be:
        - Add a storage array called `historicalTrusted`.
        - Each time toggleTrusted is set to 1, the address is `push` to the end of the storage array.
        - Create a view function that reads all addresses in the `historicalTrusted` array, checks them via `isTrusted`, and only reports the addresses that are still valid and trusted.
- **Code to Check:** Line [22](https://github.com/SolidlyV3/v3-core/blob/29c9c3c03c6d6849da7f3e13b488e3175298c9a7/contracts/UniswapV3Factory.sol#L22), Line [62](https://github.com/SolidlyV3/v3-core/blob/29c9c3c03c6d6849da7f3e13b488e3175298c9a7/contracts/UniswapV3Factory.sol#L62)
