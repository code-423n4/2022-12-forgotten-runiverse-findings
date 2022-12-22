**Overview**

Risk Rating | Number of issues
--- | ---
Gas Issues | 19

**Table of Contents:**

- [1. Using `calldata` instead of `memory`](#1-using-calldata-instead-of-memory)
- [2. Redundant checks and storage readings](#2-redundant-checks-and-storage-readings)
- [3. Refactoring the logic of a for-loop to save gas](#3-refactoring-the-logic-of-a-for-loop-to-save-gas)
- [4. Multiple accesses of a mapping/array should use a local variable cache](#4-multiple-accesses-of-a-mappingarray-should-use-a-local-variable-cache)
- [5. Caching storage values in memory](#5-caching-storage-values-in-memory)
- [6. (Proposal) Variables that should be constant/immutable](#6-proposal-variables-that-should-be-constantimmutable)
- [7. Switching between `1` and `2` instead of `0` and `1` (or `false` and `true`) is more gas efficient](#7-switching-between-1-and-2-instead-of-0-and-1-or-false-and-true-is-more-gas-efficient)
- [8. Make some variables smaller for storage packing and favorizing the user](#8-make-some-variables-smaller-for-storage-packing-and-favorizing-the-user)
- [9. Explicitly assigning a default value in storage wastes gas](#9-explicitly-assigning-a-default-value-in-storage-wastes-gas)
- [10. `++numMinted` costs less gas compared to `numMinted += 1`](#10-numminted-costs-less-gas-compared-to-numminted--1)
- [11. `++plotsMinted[uint256(plotSize)]` costs less gas compared to `plotsMinted[uint256(plotSize)] += 1`](#11-plotsminteduint256plotsize-costs-less-gas-compared-to-plotsminteduint256plotsize--1)
- [12. Increments can be unchecked in for-loops](#12-increments-can-be-unchecked-in-for-loops)
- [13. `require()` statements that check input arguments should be at the top of the function](#13-require-statements-that-check-input-arguments-should-be-at-the-top-of-the-function)
- [14. `vault` cannot realistically be `address(0)`](#14-vault-cannot-realistically-be-address0)
- [15. Unnecessary check for `msg.sender` being `address(0)`](#15-unnecessary-check-for-msgsender-being-address0)
- [16. Unchecking arithmetics operations that can't underflow/overflow](#16-unchecking-arithmetics-operations-that-cant-underflowoverflow)
- [17. Splitting `require()` statements that use `&&` saves gas](#17-splitting-require-statements-that-use--saves-gas)
- [18. Using private rather than public for constants saves gas](#18-using-private-rather-than-public-for-constants-saves-gas)
- [19. Upgrade pragma](#19-upgrade-pragma)

## 1. Using `calldata` instead of `memory`

*Notice that [c4udit](https://gist.github.com/JustDravee/4b13fe17b1b8aa7e01a2369b92719e38) doesn't mention replacing `memory` with `calldata`*

The following setters should be `external` instead of `public` and use `calldata` instead of `memory`:

```diff
File: RuniverseLandMinter.sol
- 500:     function setLocalIdOffsets(uint256[] memory _newPlotSizeLocalOffset) public onlyOwner {
+ 500:     function setLocalIdOffsets(uint256[] calldata _newPlotSizeLocalOffset) external onlyOwner {
513:     function setPlotsAvailablePerSize(
- 514:         uint256[] memory _newPlotsAvailablePerSize
+ 514:         uint256[] calldata _newPlotsAvailablePerSize
- 515:     ) public onlyOwner {
+ 515:     ) external onlyOwner {
```

Gas report:

```diff
·----------------------------------------------------|----------------------------|-------------|-----------------------------·
|                Solc version: 0.8.0                 ·  Optimizer enabled: false  ·  Runs: 200  ·  Block limit: 30000000 gas  │
·····················································|····························|·············|······························
|  Methods                                                                                                                    │
························|····························|··············|·············|·············|···············|··············
|  Contract             ·  Method                    ·  Min         ·  Max        ·  Avg        ·  # calls      ·  usd (avg)  │
························|····························|··············|·············|·············|···············|··············
- |  RuniverseLandMinter  ·  setLocalIdOffsets         ·           -  ·          -  ·      56513  ·            1  ·          -  │
+ |  RuniverseLandMinter  ·  setLocalIdOffsets         ·           -  ·          -  ·      54898  ·            1  ·          -  │
························|····························|··············|·············|·············|···············|··············
- |  RuniverseLandMinter  ·  setPlotsAvailablePerSize  ·           -  ·          -  ·      54307  ·            6  ·          -  │
+ |  RuniverseLandMinter  ·  setPlotsAvailablePerSize  ·           -  ·          -  ·      52692  ·            6  ·          -  │
·----------------------------------------------------|--------------|-------------|-------------|---------------|-------------·
```

## 2. Redundant checks and storage readings

The following can be simplified:

```diff
File: RuniverseLandMinter.sol
362:     function _mintTokensUsingTokenId(
363:         IRuniverseLand.PlotSize plotSize,
364:         uint256 tokenId,
365:         address recipient
366:     ) private {
- 367:         uint256 numPlots = 1;
368:         require(
369:             plotsMinted[uint256(plotSize)] <
370:                 plotsAvailablePerSize[uint256(plotSize)],
371:             "All plots of that size minted"
372:         );
- 373:         require(
- 374:             plotsMinted[uint256(plotSize)] + numPlots <=
- 375:                 plotsAvailablePerSize[uint256(plotSize)],
- 376:             "Trying to mint too many plots"
- 377:         );
378: 
379:         plotsMinted[uint256(plotSize)] += 1;
380: 
381: 
382:         runiverseLand.mintTokenId(recipient, tokenId, plotSize);
383:     }
```

Indeed, here, the second check is logically equivalent to the 1st as `numPlots == 1`.
It wouldn't make much sense either to say "Trying to mint too many plots" when the number of plots being minted is constantly 1.
The message "All plots of that size minted" is the only one relevant.

Gas saved: **around 200**

## 3. Refactoring the logic of a for-loop to save gas

The following can be optimized:

```solidity
File: RuniverseLandMinter.sol
323:     function _mintTokens(
...    
338:         for (uint256 i = 0; i < numPlots; i++) {
339: 
340:             uint256 tokenId = ownerGetNextTokenId(plotSize);            
341:             plotsMinted[uint256(plotSize)] += 1;          
342:                
343:             runiverseLand.mintTokenId(recipient, tokenId, plotSize);
344:         }        
345:     }
```

Indeed, here, `ownerGetNextTokenId(plotSize)` is a `private` function only called in this for-loop:

```solidity
File: RuniverseLandMinter.sol
399:     function ownerGetNextTokenId(IRuniverseLand.PlotSize plotSize) private view returns (uint256) {
400:         uint256 globalCounter = plotsMinted[0] + plotsMinted[1] + plotsMinted[2] + plotsMinted[3] + plotsMinted[4] + plotGlobalOffset;
401:         uint256 localCounter  = plotsMinted[uint256(plotSize)] + plotSizeLocalOffset[uint256(plotSize)];
402:         require( localCounter <= 4294967295, "Local index overflow" );
403:         require( uint256(plotSize) <= 255, "Plot index overflow" );
404:         
405:         return (globalCounter<<40) + (localCounter<<8) + uint256(plotSize);
406:     }
```

At each iteration, multiple storage readings are happening in `ownerGetNextTokenId()`, and `plotsMinted` gets written every time.

Several of those can be taken out of the for-loop and writing into `plotsMinted` can actually be done after the for-loop:

```solidity
File: RuniverseLandMinter.sol
323:     function _mintTokens(
...    
343:         uint256 globalCounter = plotsMinted[0] +
344:             plotsMinted[1] +
345:             plotsMinted[2] +
346:             plotsMinted[3] +
347:             plotsMinted[4] +
348:             plotGlobalOffset;
349: 
350:         uint256 localCounter = plotsMinted[uint256(plotSize)] +
351:             plotSizeLocalOffset[uint256(plotSize)];
352: 
353:         require(uint256(plotSize) <= 255, "Plot index overflow");
354: 
355:         bool isFirstFourPlots = uint256(plotSize) < 5;
356: 
357:         for (uint256 i = 0; i < numPlots; i++) {
358:             require(localCounter <= 4294967295, "Local index overflow");
359: 
360:             uint256 tokenId = (globalCounter << 40) +
361:                 (localCounter << 8) +
362:                 uint256(plotSize);
363: 
364:             runiverseLand.mintTokenId(recipient, tokenId, plotSize);
365: 
366:             if (isFirstFourPlots) {
367:                 ++globalCounter;
368:             }
369: 
370:             ++localCounter;
371:         }
372:         
373:         plotsMinted[uint256(plotSize)] += numPlots;
```

Gas report (keep in mind that the savings could look more massive with more tests):

```diff
·----------------------------------------------------|----------------------------|-------------|-----------------------------·
|                Solc version: 0.8.0                 ·  Optimizer enabled: false  ·  Runs: 200  ·  Block limit: 30000000 gas  │
·····················································|····························|·············|······························
|  Methods                                                                                                                    │
························|····························|··············|·············|·············|···············|··············
|  Contract             ·  Method                    ·  Min         ·  Max        ·  Avg        ·  # calls      ·  usd (avg)  │
························|····························|··············|·············|·············|···············|··············
- |  RuniverseLandMinter  ·  mint                      ·           -  ·          -  ·     161341  ·            1  ·          -  │
+ |  RuniverseLandMinter  ·  mint                      ·           -  ·          -  ·     161319  ·            1  ·          -  │
························|····························|··············|·············|·············|···············|··············
- |  RuniverseLandMinter  ·  mintlistMint              ·      121584  ·     193104  ·     157344  ·            2  ·          -  │
+ |  RuniverseLandMinter  ·  mintlistMint              ·      121562  ·     193082  ·     157322  ·            2  ·          -  │
························|····························|··············|·············|·············|···············|··············
- |  RuniverseLandMinter  ·  ownerMint                 ·      103410  ·     154710  ·     122648  ·            8  ·          -  │
+ |  RuniverseLandMinter  ·  ownerMint                 ·      103388  ·     154688  ·     122626  ·            8  ·          -  │
```

## 4. Multiple accesses of a mapping/array should use a local variable cache

Caching a mapping's value in a local `storage` or `calldata` variable when the value is accessed multiple times saves **~42 gas per access** due to not having to perform the same offset calculation every time.

Affected code:

- `RuniverseLandMinter.sol#mintlistMintedPerSize[msg.sender][uint256(plotSize)]`:

```diff
+ 247:         mapping(uint256 => uint256) storage mintedPerSize = mintlistMintedPerSize[msg.sender];
248:         require(
- 249:             mintlistMintedPerSize[msg.sender][uint256(plotSize)] + numPlots <=
+ 249:             mintedPerSize[uint256(plotSize)] + numPlots <=
250:                 claimedMaxPlots, // this is verified by the merkle proof
251:             "Minting more than allowed"
252:         );
- 253:         mintlistMintedPerSize[msg.sender][uint256(plotSize)] += numPlots;
+ 253:         mintedPerSize[uint256(plotSize)] += numPlots;
```

- `RuniverseLandMinter.sol#claimlistMintedPerSize[msg.sender][uint256(plotSize)]`:

```diff
+ 287:         mapping(uint256 => uint256) storage mintedPerSize = claimlistMintedPerSize[msg.sender];
288:         require(
- 289:             claimlistMintedPerSize[msg.sender][uint256(plotSize)] + numPlots <=
+ 289:             mintedPerSize[uint256(plotSize)] + numPlots <=
290:                 claimedMaxPlots, // this is verified by the merkle proof
291:             "Claiming more than allowed"
292:         );
- 293:         claimlistMintedPerSize[msg.sender][uint256(plotSize)] += numPlots;
+ 293:         mintedPerSize[uint256(plotSize)] += numPlots;
```

## 5. Caching storage values in memory

The code can be optimized by minimizing the number of SLOADs.

SLOADs are expensive (100 gas after the 1st one) compared to MLOADs/MSTOREs (3 gas each). Storage values read multiple times should instead be cached in memory the first time (costing 1 SLOAD) and then read from this cache to avoid multiple SLOADs.

- `contracts/ERC721Vestable.sol#_beforeTokenTransfer()`: `lastVestingGlobalId`, `vestingStart`, `vestingEnd`

```solidity
  38          if (
  39              vestingEnabled &&
  40              from != address(0) && // minting
  41:             globalId <= lastVestingGlobalId && //@audit gas SLOAD 1 (lastVestingGlobalId)
  42:             block.timestamp < vestingEnd //@audit gas SLOAD 1 (vestingEnd)
  43          ) {
  44:             uint256 vestingDuration = vestingEnd - vestingStart; //@audit gas SLOAD 2 (vestingEnd) & SLOAD 1 (vestingStart)
  45              uint256 chunk = vestingDuration / lastVestingGlobalId;//@audit gas SLOAD 2 (lastVestingGlobalId)
  46              require(
  47:                 block.timestamp >= (chunk * globalId) + vestingStart,//@audit gas SLOAD 2 (vestingStart)
  48                  "Not vested"
  49              );
```

- `contracts/ERC721Vestable.sol#vestsAt()`: `vestingStart`

```solidity
  63      function vestsAt(uint256 tokenId) public view returns (uint256) {
  64          uint256 globalId = getGlobalId(tokenId);
  65:         uint256 vestingDuration = vestingEnd - vestingStart;//@audit gas SLOAD 1 (vestingStart)
  66          uint256 chunk = vestingDuration / lastVestingGlobalId;
  67:         return (chunk * globalId) + vestingStart;//@audit gas SLOAD 2 (vestingStart)
  68      }
```

- `contracts/RuniverseLand.sol#mintTokenId()`: `numMinted`

```solidity
  93:         require(numMinted < MAX_SUPPLY, "All land has been minted");//@audit gas SLOAD 1 (numMinted)
...
  98:         numMinted += 1;//@audit gas SLOAD 2 (numMinted, could've used numMinted = cachedValue + 1)
```

## 6. (Proposal) Variables that should be constant/immutable

There are some variables that are very unlikely to change

```solidity
File: ERC721Vestable.sol
16:     /// @notice the tokens from 0 to lastVestedTokenId will vest over time
17:     uint256 public lastVestingGlobalId = 10924;
18: 
19:     /// @notice the time the vesting started
20:     uint256 public vestingStart = 1671840000; // Dec 24th, 2022
21: 
22:     /// @notice the time the vesting ends
23:     uint256 public vestingEnd = 1734998400; // Dec 24th, 2024
```

Consider deleting their setters and marking them as immutable to save a massive amount of gas (**20 000 gas per constant**)

## 7. Switching between `1` and `2` instead of `0` and `1` (or `false` and `true`) is more gas efficient

`SSTORE` from 0 to 1 (or any non-zero value) costs 20000 gas.
`SSTORE` from 1 to 2 (or any other non-zero value) costs 5000 gas.

By storing the original value once again, a refund is triggered (<https://eips.ethereum.org/EIPS/eip-2200>).

Since refunds are capped to a percentage of the total transaction's gas, it is best to keep them low, to increase the likelihood of the full refund coming into effect.

Therefore, switching between 1, 2 instead of 0, 1 will be more gas efficient.

See: <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/86bd4d73896afcb35a205456e361436701823c7a/contracts/security/ReentrancyGuard.sol#L29-L33>

Affected code:

```solidity
contracts/ERC721Vestable.sol:
   14:     bool public vestingEnabled = true;  //@audit-issue gas should be a switch between 1-2
```

## 8. Make some variables smaller for storage packing and favorizing the user

If the following variables can't be made constant, they should still be made smaller for storage packing.

Indeed, a timestamp of type `uint32` has a max timestamp corresponding to year `2106` (`4294967295`) and `uint48` has a max-year of around `9,000,000`. Also, `lastVestingGlobalId` is unlikely to be bigger than `uint32`'s max. Therefore, using `uint256` is too much.
As the 3 states variables in `ERC721Vestable.sol` are unlikely to change much during the contract's lifespan and state-variable-writing being actually the concern of the `owner` instead of the users, I would recommend the following layout:

- File: ERC721Vestable.sol:

```diff
12: abstract contract ERC721Vestable is ERC721 {
13:     /// @notice master switch for vesting
14:     bool public vestingEnabled = true;
15: 
16:     /// @notice the tokens from 0 to lastVestedTokenId will vest over time
- 17:     uint256 public lastVestingGlobalId = 10924;
+ 17:     uint32 public lastVestingGlobalId = 10924;
18: 
19:     /// @notice the time the vesting started
- 20:     uint256 public vestingStart = 1671840000; // Dec 24th, 2022
+ 20:     uint32 public vestingStart = 1671840000; // Dec 24th, 2022
21: 
22:     /// @notice the time the vesting ends
- 23:     uint256 public vestingEnd = 1734998400; // Dec 24th, 2024
+ 23:     uint32 public vestingEnd = 1734998400; // Dec 24th, 2024
```

Similar suggestion here:

- File: RuniverseLandMinter.sol:

```diff
19:     /// @notice Address to the vault where we can withdraw
20:     address payable public vault;
+ 20:     uint32 public publicMintStartTime = type(uint32).max;
+ 20:     uint32 public mintlistStartTime = type(uint32).max;
+ 20:     uint32 public claimsStartTime = type(uint32).max;
...
- 48:     uint256 public publicMintStartTime = type(uint256).max;
- 49:     uint256 public mintlistStartTime = type(uint256).max;
- 50:     uint256 public claimsStartTime = type(uint256).max;
```

Keep in mind that the setter-functions will have to be adapted too. While these setters will be slightly more expensive (the `owner` pays for the overhead), the functions used by the users will be less expensive:

```diff
·----------------------------------------------------|----------------------------|-------------|-----------------------------·
|                Solc version: 0.8.0                 ·  Optimizer enabled: false  ·  Runs: 200  ·  Block limit: 30000000 gas  │
·····················································|····························|·············|······························
|  Methods                                                                                                                    │
························|····························|··············|·············|·············|···············|··············
|  Contract             ·  Method                    ·  Min         ·  Max        ·  Avg        ·  # calls      ·  usd (avg)  │
························|····························|··············|·············|·············|···············|··············
- |  RuniverseLand        ·  transferFrom              ·       48157  ·      67947  ·      56805  ·            6  ·          -  │
+ |  RuniverseLand        ·  transferFrom              ·       44341  ·      66039  ·      52223  ·            6  ·          -  │
························|····························|··············|·············|·············|···············|··············
- |  RuniverseLandMinter  ·  claimlistMint             ·      116648  ·     185048  ·     150848  ·            2  ·          -  │
+ |  RuniverseLandMinter  ·  claimlistMint             ·      116608  ·     185008  ·     150808  ·            2  ·          -  │
·----------------------------------------------------|--------------|-------------|-------------|---------------|-------------·
```

## 9. Explicitly assigning a default value in storage wastes gas

This is a useless storage writing as it assigns the default value:

```diff
contracts/RuniverseLand.sol:
-   43:     uint256 public numMinted = 0; //@audit-issue gas: explicit 0 not necessary
+   43:     uint256 public numMinted;
```

Gas report:

```diff
|  Deployments                                       ·                                          ·  % of limit   ·             │
·····················································|··············|·············|·············|···············|··············
- |  RuniverseLand                                     ·           -  ·          -  ·    4243099  ·       14.1 %  ·          -  │
+ |  RuniverseLand                                     ·           -  ·          -  ·    4240825  ·       14.1 %  ·          -  │
```

## 10. `++numMinted` costs less gas compared to `numMinted += 1`

*This one isn't in a for-loop and isn't covered by [c4udit](https://gist.github.com/JustDravee/4b13fe17b1b8aa7e01a2369b92719e38#GAS-8)*

```diff
contracts/RuniverseLand.sol:
-   98:         numMinted += 1;
+   98:         ++numMinted; 
```

## 11. `++plotsMinted[uint256(plotSize)]` costs less gas compared to `plotsMinted[uint256(plotSize)] += 1`

*This one isn't covered by [c4udit](https://gist.github.com/JustDravee/4b13fe17b1b8aa7e01a2369b92719e38#GAS-8)*

```diff
File: RuniverseLandMinter.sol
- 341:             plotsMinted[uint256(plotSize)] += 1;          
+ 341:             ++plotsMinted[uint256(plotSize)];          
- 379:             plotsMinted[uint256(plotSize)] += 1;          
+ 379:             ++plotsMinted[uint256(plotSize)];          
```

## 12. Increments can be unchecked in for-loops

*This one isn't covered by [c4udit](https://gist.github.com/JustDravee/4b13fe17b1b8aa7e01a2369b92719e38)*

In Solidity 0.8+, there's a default overflow check on unsigned integers. It's possible to uncheck this in for-loops and save some gas at each iteration, but at the cost of some code readability, as this uncheck cannot be made inline.  
  
[ethereum/solidity#10695](https://github.com/ethereum/solidity/issues/10695)

Consider wrapping with an `unchecked` block here (around **25 gas saved** per instance):

```solidity
RuniverseLandMinter.sol:338:        for (uint256 i = 0; i < numPlots; i++) {
```

The change would be:  
  
```diff
File: RuniverseLandMinter.sol
- 338:         for (uint256 i = 0; i < numPlots; i++) {
+ 338:         for (uint256 i = 0; i < numPlots;) {
339: 
340:             uint256 tokenId = ownerGetNextTokenId(plotSize);            
341:             plotsMinted[uint256(plotSize)] += 1;          
342:                
343:             runiverseLand.mintTokenId(recipient, tokenId, plotSize);
+ 343:             unchecked { ++i; }
344:         }        
```

The risk of overflow is non-existent for `uint256` here.

## 13. `require()` statements that check input arguments should be at the top of the function

Checks that involve constants should come before checks that involve state variables, function calls, and calculations. By doing these checks first, the function is able to revert before wasting a SLOAD (**2100 gas** for the 1st one) in a function that may ultimately revert in the unhappy case.

```diff
File: RuniverseLandMinter.sol
528:     function setPrices(uint256[] calldata _newPrices) public onlyOwner {
- 529:         require(!mintlistStarted(), "Can't change during mint");
+ 529:         require(_newPrices.length == 5, "must set exactly 5 prices");
- 530:         require(_newPrices.length == 5, "must set exactly 5 prices");
+ 530:         require(!mintlistStarted(), "Can't change during mint");
531:         plotPrices = _newPrices;
532:     }
```

Notice that this good practice is already applied for `setLocalIdOffsets`:

```solidity
File: RuniverseLandMinter.sol
500:     function setLocalIdOffsets(uint256[] memory _newPlotSizeLocalOffset) public onlyOwner {
501:         require(
502:             _newPlotSizeLocalOffset.length == 5,
503:             "must set exactly 5 numbers"
504:         );
505:         require(!mintlistStarted(), "Can't change during mint");
506:         plotSizeLocalOffset = _newPlotSizeLocalOffset;
507:     }
```

## 14. `vault` cannot realistically be `address(0)`

`vault` is set in the constructor as `msg.sender`:

```solidity
File: RuniverseLandMinter.sol
76:     constructor(IRuniverseLand _runiverseLand) {
77:         setRuniverseLand(_runiverseLand);
78:         setVaultAddress(payable(msg.sender));
79:     }
```

And then, `setVaultAddress()` can only be called by the owner:

```solidity
File: RuniverseLandMinter.sol
480:     function setVaultAddress(address payable _newVaultAddress)
481:         public
482:         onlyOwner
483:     {
484:         vault = _newVaultAddress;
485:     }
```

It seems extremely unlikely that the `owner` (who is, by the way, also the `vault`) would just happen to set `vault` to `address(0)`. Even if done by mistake, it can be fixed fast.
Therefore, calling `withdraw` or `withdrawAll`, which are 2 `onlyOwner` functions, while having a `vault == address(0)`, is way too unlikely:

```diff
File: RuniverseLandMinter.sol
538:     function withdraw(uint256 _amount) public onlyOwner {
- 539:         require(address(vault) != address(0), "no vault");
540:         vault.sendValue(_amount);
541:     }
542: 
543:     /**
544:      * @notice Withdraw all the funds to the vault using sendValue     
545:      */
546:     function withdrawAll() public onlyOwner {
- 547:         require(address(vault) != address(0), "no vault");
548:         vault.sendValue(address(this).balance);
549:     }
```

If what's feared is a compromise, then just add the check in the setter, which should be called less often that the withdraw functions:

```diff
File: RuniverseLandMinter.sol
480:     function setVaultAddress(address payable _newVaultAddress)
481:         public
482:         onlyOwner
483:     {
+ 484:         require(address(_newVaultAddress) != address(0), "no vault");
484:         vault = _newVaultAddress;
485:     }
```

## 15. Unnecessary check for `msg.sender` being `address(0)`

Unless there's a way to own `address(0)` (in which case, please DM), these checks should be deleted:

```solidity
contracts/RuniverseLand.sol:
  221:         require(address(msg.sender) != address(0), "req sender");

contracts/RuniverseLandMinter.sol:
  557:         require(address(msg.sender) != address(0), "req sender");
```

## 16. Unchecking arithmetics operations that can't underflow/overflow

Solidity version 0.8+ comes with implicit overflow and underflow checks on unsigned integers. When an overflow or an underflow isn't possible (as an example, when a comparison is made before the arithmetic operation), some gas can be saved by using an `unchecked` block: <https://docs.soliditylang.org/en/v0.8.10/control-structures.html#checked-or-unchecked-arithmetic>

As the following only concerns substractions with `owner` supplied value, it's not a stretch to consider that they are safe/trusted enough be wrapped with an `unchecked` block (around **25 gas saved** per instance):

```solidity
ERC721Vestable.sol:44:            uint256 vestingDuration = vestingEnd - vestingStart;
ERC721Vestable.sol:65:        uint256 vestingDuration = vestingEnd - vestingStart;
RuniverseLandMinter.sol:168:        plotsAvailableBySize[0] = plotsAvailablePerSize[0] - plotsMinted[0];
RuniverseLandMinter.sol:169:        plotsAvailableBySize[1] = plotsAvailablePerSize[1] - plotsMinted[1];
RuniverseLandMinter.sol:170:        plotsAvailableBySize[2] = plotsAvailablePerSize[2] - plotsMinted[2];
RuniverseLandMinter.sol:171:        plotsAvailableBySize[3] = plotsAvailablePerSize[3] - plotsMinted[3];
RuniverseLandMinter.sol:172:        plotsAvailableBySize[4] = plotsAvailablePerSize[4] - plotsMinted[4];
```

## 17. Splitting `require()` statements that use `&&` saves gas

See [this issue](https://github.com/code-423n4/2022-01-xdefi-findings/issues/128) which describes the fact that there is a larger deployment gas cost, but with enough runtime calls, the change ends up being cheaper

Affected code (saving around **3 gas** per instance):

```solidity
RuniverseLandMinter.sol:212:        require(numPlots > 0 && numPlots <= 20, "Mint from 1 to 20 plots");        
RuniverseLandMinter.sol:230:        require(numPlots > 0 && numPlots <= 20, "Mint from 1 to 20 plots");
```

## 18. Using private rather than public for constants saves gas

If needed, the value can be read from the verified contract source code. Savings are due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table.

```solidity
RuniverseLand.sol:40:    uint256 public constant MAX_SUPPLY = 70000;
RuniverseLand.sol:57:    string public constant R = "I should like to save the Shire, if I could"; 
```

## 19. Upgrade pragma

Using newer compiler versions and the optimizer give gas optimizations. Also, additional safety checks are available for free.

The advantages here are:

- **Low level inliner** (>= 0.8.2): Cheaper runtime gas (especially relevant when the contract has small functions).
- **Optimizer improvements in packed structs** (>= 0.8.3)
- **Custom errors** (>= 0.8.4): cheaper deployment cost and runtime cost. *Note*: the runtime cost is only relevant when the revert condition is met. In short, replace revert strings by custom errors.
- **Contract existence checks** (>= 0.8.10): external calls skip contract existence checks if the external call has a return value

Consider upgrading here :

```solidity
ERC721Vestable.sol:5:pragma solidity ^0.8.0;
IRuniverseLand.sol:2:pragma solidity ^0.8.0;
RuniverseLand.sol:8:pragma solidity ^0.8.0;
RuniverseLandMinter.sol:2:pragma solidity ^0.8.0;
```
