**Overview**
Risk Rating | Number of issues
--- | ---
Low Risk | 11
Non-Critical Risk | 8

**Table of Contents**

- [1. Low Risk Issues](#1-low-risk-issues)
  - [1.1. Either `claimlistMint` shouldn't be payable, or it should call `_mintTokensCheckingValue()` instead of `_mintTokens`](#11-either-claimlistmint-shouldnt-be-payable-or-it-should-call-_minttokenscheckingvalue-instead-of-_minttokens)
  - [1.2. `withdrawAll` being `payable` for no reason](#12-withdrawall-being-payable-for-no-reason)
  - [1.3. Consider starting `tokenId` at 1 instead of 0](#13-consider-starting-tokenid-at-1-instead-of-0)
  - [1.4. Returning ETH at the end of the call would be more user-friendly than asking an exact amount](#14-returning-eth-at-the-end-of-the-call-would-be-more-user-friendly-than-asking-an-exact-amount)
  - [1.5. Missing events on setters](#15-missing-events-on-setters)
  - [1.6. It's better to emit after all processing is done](#16-its-better-to-emit-after-all-processing-is-done)
  - [1.7. Missing input validation for `numPlots`](#17-missing-input-validation-for-numplots)
  - [1.8. Events not indexed](#18-events-not-indexed)
  - [1.9. Use a 2-step ownership transfer pattern](#19-use-a-2-step-ownership-transfer-pattern)
  - [1.10. Mismatch between interface and implementation](#110-mismatch-between-interface-and-implementation)
  - [1.11. Passing the leaf as an argument is redundant as it contains no additional information](#111-passing-the-leaf-as-an-argument-is-redundant-as-it-contains-no-additional-information)
- [2. Non-Critical Issues](#2-non-critical-issues)
  - [2.1. Non-traditional Use of ReentrancyGuard](#21-non-traditional-use-of-reentrancyguard)
  - [2.2. Duplicated code: Hardcoded strings having multiple occurrences should be declared as `constant`](#22-duplicated-code-hardcoded-strings-having-multiple-occurrences-should-be-declared-as-constant)
  - [2.3. Typos and syntax](#23-typos-and-syntax)
  - [2.4. Remove unused function](#24-remove-unused-function)
  - [2.5. Upgrade to Solidity version `0.8.12`  and use `string.concat()` or `bytes.concat()`](#25-upgrade-to-solidity-version-0812--and-use-stringconcat-or-bytesconcat)
  - [2.6. Use named returns where relevant](#26-use-named-returns-where-relevant)
  - [2.7. Magic numbers](#27-magic-numbers)
  - [2.8. Non-library/interface files should use fixed compiler versions, not floating ones](#28-non-libraryinterface-files-should-use-fixed-compiler-versions-not-floating-ones)

# 1. Low Risk Issues

## 1.1. Either `claimlistMint` shouldn't be payable, or it should call `_mintTokensCheckingValue()` instead of `_mintTokens`

`_mintTokens()` is a private function used in `ownerMint()` and doesn't forward any `msg.value`.
`_mintTokensCheckingValue()` is a private function used in `mintlistMint()` and `mint()`, and checks `msg.value`.

The situation for `claimlistMint()` is ambiguous as the function is `payable` but uses `_mintTokens()`. There are no checks for `msg.value` in `claimlistMint()` and the final `runiverseLand.mintTokenId()` call doesn't forward any `msg.value` (`runiverseLand.mintTokenId` isn't even `payable`)

For this ambiguous situation, it depends on how private or privileged the claimers in the claimlist are supposed to be, but one of those 2 things need to be done:

- Either `claimlistMint()` shouldn't be payable (no reasons in the current flow)
- Or `_mintTokensCheckingValue()` should be used instead of `_mintTokens` in `claimlistMint()`, in which case this issue's severity should be upgraded

## 1.2. `withdrawAll` being `payable` for no reason

There's no reason for a withdraw function to also transmit ETH to the contract. Consider removing the `payable` keyword:

```diff
contracts/RuniverseLand.sol:
-  209:     function withdrawAll() public payable onlyOwner { //@audit-issue payable for no reason and it'll leave funds inside
+  209:     function withdrawAll() public onlyOwner {
```

## 1.3. Consider starting `tokenId` at 1 instead of 0

It is a trend to start tokenIds with 1, as this prevents some integration issues with front-ends and is generally nicer

```diff
contracts/RuniverseLand.sol:
-   43:     uint256 public numMinted = 0;
+   43:     uint256 public numMinted = 1;
   77:         uint256 tokenId = numMinted; //@audit this is better when started with 1
````

Do not forget to fix `totalSupply()`:

```diff
File: RuniverseLand.sol
145:     function totalSupply() public view returns (uint256) {
- 146:         return numMinted;
+ 146:         return numMinted - 1;
147:     }
```

Also fix the following check:

```diff
File: RuniverseLand.sol
- 93:         require(numMinted < MAX_SUPPLY, "All land has been minted");
+ 93:         require(numMinted <= MAX_SUPPLY, "All land has been minted"); 
```

## 1.4. Returning ETH at the end of the call would be more user-friendly than asking an exact amount

This check feels restrictive:

```Solidity
contracts/RuniverseLandMinter.sol:
  310:             msg.value == plotPrices[uint256(plotSize)] * numPlots,  //@audit-issue Better just send back the excess tokens at the end
```

There might even be issues with the front-end due to Ether's 18 decimals & JavaScript's precision.
I'd recommend authorizing an excess `msg.value` and returning the excess to `msg.sender` at the end of the call

## 1.5. Missing events on setters

There's only 1 event in the whole project. Consider adding some in the setters so that users can be notified when a change is occurring.

## 1.6. It's better to emit after all processing is done

This would prevent polluting dashboards:

```solidity
File: RuniverseLand.sol
088:     function mintTokenId(
...
099:         emit LandMinted(recipient, tokenId, size); //@audit-issue should emit at the end to avoid pollution
100:         
101:         _mint(recipient, tokenId);
102:     }
```

## 1.7. Missing input validation for `numPlots`

`numPlots == 0` is a valid value: <https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLandMinter.sol#L264-L295>

## 1.8. Events not indexed

When there are 3 or more arguments, it's encouraged to index your events.

```solidity
IRuniverseLand.sol:18:    event LandMinted(address to, uint256 tokenId, IRuniverseLand.PlotSize size);
```

## 1.9. Use a 2-step ownership transfer pattern

*ToB report about it : <https://github.com/trailofbits/publications/blob/master/reviews/LooksRare.pdf>*

Contracts inheriting from OpenZeppelin's libraries have the default `transferOwnership()` function (a one-step process). It's possible that the `onlyOwner` role mistakenly transfers ownership to a wrong address, resulting in a loss of the `onlyOwner` role.
Consider overriding the default `transferOwnership()` function to first nominate an address as the `pendingOwner` and implementing an `acceptOwnership()` function which is called by the `pendingOwner` to confirm the transfer.

```solidity
RuniverseLand.sol:11:import "@openzeppelin/contracts/access/Ownable.sol";
RuniverseLand.sol:32:    Ownable,
```

```solidity
RuniverseLandMinter.sol:6:import "@openzeppelin/contracts/access/Ownable.sol";
RuniverseLandMinter.sol:13:contract RuniverseLandMinter is Ownable, ReentrancyGuard {
```

## 1.10. Mismatch between interface and implementation

The following are `external` in the interface:

```solidity
File: IRuniverseLand.sol
20:     function mint(address recipient, PlotSize size) external returns (uint256);//@audit mismatch with impl
21: 
22:     function mintTokenId(
23:         address recipient,
24:         uint256 tokenId,
25:         PlotSize size
26:     ) external; //@audit mismatch with impl
```

But `public` in the implementation:

```solidity
contracts/RuniverseLand.sol:
  72:     function mint(address recipient, PlotSize size)
  73          public

  88:     function mintTokenId(
  89          address recipient,
  90          uint256 tokenId, 
  91          PlotSize size
  92      ) public override nonReentrant { 
```

## 1.11. Passing the leaf as an argument is redundant as it contains no additional information

```diff
File: RuniverseLandMinter.sol
184:     function mintlisted(
185:         address _who,
- 186:         bytes32 _leaf,
187:         bytes32[] calldata _merkleProof
188:     ) external view returns (bool) {
189:         bytes32 node = keccak256(abi.encodePacked(_who));
190:         
- 191:         if (node != _leaf) return false;
```

# 2. Non-Critical Issues

## 2.1. Non-traditional Use of ReentrancyGuard

The [NatSpec documentation on ReentrancyGuard.sol](https://github.com/compound-finance/compound-protocol/blob/f385d71983ae5c5799faae9b2dfea43e5cf75262/contracts/ReentrancyGuard.sol#L6-L7) states:

> If you mark a function nonReentrant, you should also mark it external.

However, the following functions have the nonReentrant modifier and are NOT external:

```solidity
RuniverseLand.sol:92:    ) public override nonReentrant {
RuniverseLandMinter.sol:208:        payable 
RuniverseLandMinter.sol:209:        nonReentrant
RuniverseLandMinter.sol:228:    ) public payable nonReentrant {
RuniverseLandMinter.sol:269:    ) public payable nonReentrant {
```

As a reminder, I do suggest deleting `RuniverseLand.sol#mint()` in another submission, so `mintTokenId()` can also be declared as external just like it is in the interface `IRuniverseLand.sol`

## 2.2. Duplicated code: Hardcoded strings having multiple occurrences should be declared as `constant`

```solidity
RuniverseLandMinter.sol:229:        require(mintlistStarted(), "Mint not started");
RuniverseLandMinter.sol:211:        require(publicStarted(), "Mint not started");
```

```solidity
RuniverseLandMinter.sol:212:        require(numPlots > 0 && numPlots <= 20, "Mint from 1 to 20 plots");        
RuniverseLandMinter.sol:230:        require(numPlots > 0 && numPlots <= 20, "Mint from 1 to 20 plots");
```

```solidity
RuniverseLandMinter.sol:245:            "Invalid proof."
RuniverseLandMinter.sol:285:            "Invalid proof."
```

```solidity
RuniverseLandMinter.sol:331:            "All plots of that size minted"
RuniverseLandMinter.sol:371:            "All plots of that size minted"
```

```solidity
RuniverseLandMinter.sol:336:            "Trying to mint too many plots"
RuniverseLandMinter.sol:376:            "Trying to mint too many plots"
```

```solidity
RuniverseLandMinter.sol:492:        require(!mintlistStarted(), "Can't change during mint");
RuniverseLandMinter.sol:505:        require(!mintlistStarted(), "Can't change during mint");
RuniverseLandMinter.sol:529:        require(!mintlistStarted(), "Can't change during mint");
```

```solidity
RuniverseLandMinter.sol:503:            "must set exactly 5 numbers"
RuniverseLandMinter.sol:519:            "must set exactly 5 numbers"
RuniverseLandMinter.sol:530:        require(_newPrices.length == 5, "must set exactly 5 prices");
```

```solidity
RuniverseLandMinter.sol:539:        require(address(vault) != address(0), "no vault");
RuniverseLandMinter.sol:547:        require(address(vault) != address(0), "no vault");
```

## 2.3. Typos and syntax

- Grammar:

```diff
File: RuniverseLandMinter.sol
- 202:      * @dev public method  for public minting. //@audit-issue extra space
+ 202:      * @dev public method for public minting.
- 516:         //msc: should we make sure all the numbres are equal or greater? //@audit-issue typo numbres
+ 516:         //msc: should we make sure all the numbers are equal or greater? 

File: RuniverseLandMinter.sol
- 106:      * @dev returns how many plots were avialable since the begining.
+ 106:      * @dev returns how many plots were available since the beginning.

File: ERC721Vestable.sol
- 54:      * @notice returns true if a tokenId has besting property.
+ 54:      * @notice returns true if a tokenId has vesting property.
```

- Sense

```diff
File: ERC721Vestable.sol
103:     /**
- 104:      * @notice set the new vesting start time
+ 104:      * @notice set the new vesting end time
105:      */
106:     function _setVestingEnd(uint256 _newVestingEnd) internal virtual {
```

## 2.4. Remove unused function

The following is never used:

```diff
File: RuniverseLand.sol
133:     /**
134:      * @dev Returns the base uri of the token.
135:      * @return _baseURI string prefix uri. 
136:      */
137:     function _baseURI() internal view virtual override returns (string memory) {
138:         return baseTokenURI;
139:     }
```

## 2.5. Upgrade to Solidity version `0.8.12`  and use `string.concat()` or `bytes.concat()`

Solidity version 0.8.12 introduces `string.concat()` (vs `abi.encodePacked(<str>,<str>)`)

```solidity
contracts/RuniverseLand.sol:
  121:         return string(abi.encodePacked(baseTokenURI, tokenId.toString()));
```

## 2.6. Use named returns where relevant

- `File: RuniverseLandMinter.sol#getTotalMintedLands()`

```diff
- 134:     function getTotalMintedLands() public view returns (uint256) {
+ 134:     function getTotalMintedLands() public view returns (uint256 totalMintedLands) {
- 135:         uint256 totalMintedLands;
136:         totalMintedLands =  plotsMinted[0] +
137:                             plotsMinted[1] +
138:                             plotsMinted[2] +                             
139:                             plotsMinted[3] +
140:                             plotsMinted[4];
- 141:         return totalMintedLands;                                                        
142:     }
```

- `File: RuniverseLandMinter.sol#getTotalMintedLandsBySize()`

```diff
- 149:     function getTotalMintedLandsBySize() public view returns (uint256[] memory) {
+ 149:     function getTotalMintedLandsBySize() public view returns (uint256[5] memory plotsMintedBySize) {
- 150:         uint256[] memory plotsMintedBySize = new uint256[](5);
151: 
152:         plotsMintedBySize[0] = plotsMinted[0];
153:         plotsMintedBySize[1] = plotsMinted[1];
154:         plotsMintedBySize[2] = plotsMinted[2];
155:         plotsMintedBySize[3] = plotsMinted[3];
156:         plotsMintedBySize[4] = plotsMinted[4];
157: 
- 158:         return plotsMintedBySize;
159:     }
```

## 2.7. Magic numbers

Replace magic numbers with constants:

```solidity
File: RuniverseLandMinter.sol
402:         require( localCounter <= 4294967295, "Local index overflow" );
403:         require( uint256(plotSize) <= 255, "Plot index overflow" );
```

## 2.8. Non-library/interface files should use fixed compiler versions, not floating ones

```solidity
ERC721Vestable.sol:5:pragma solidity ^0.8.0;
IRuniverseLand.sol:2:pragma solidity ^0.8.0;
RuniverseLand.sol:8:pragma solidity ^0.8.0;
RuniverseLandMinter.sol:2:pragma solidity ^0.8.0;
```
