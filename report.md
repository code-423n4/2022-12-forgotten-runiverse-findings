---
sponsor: "Forgotten Runiverse"
slug: "2022-12-forgotten-runiverse"
date: "2023-12-01"
title: "Forgotten Runiverse - Versus contest"
findings: "https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues"
contest: 198
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Forgotten Runiverse smart contract system written in Solidity. The audit took place between December 20—December 22 2022.

## Wardens

In Code4rena's Versus audits, the competition is limited to a small group of wardens; for this audit, 4 wardens contributed reports:

  1. [Dravee](https://twitter.com/BowTiedDravee)
  2. Lambda
  3. cccz
  4. [hansfriese](https://twitter.com/hansfriese)

This audit was judged by [Alex the Entreprenerd](https://code4rena.com/@GalloDaSballo).

Final report assembled by [liveactionllama](https://twitter.com/liveactionllama).

# Summary

The C4 analysis yielded an aggregated total of 7 unique vulnerabilities. Of these vulnerabilities, 0 received a risk rating in the category of HIGH severity and 7 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 4 reports detailing issues with a risk rating of LOW severity or non-critical. There was also 1 report recommending gas optimizations.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Forgotten Runiverse Versus repository](https://github.com/code-423n4/2022-12-forgotten-runiverse), and is composed of 4 smart contracts written in the Solidity programming language and includes 516 lines of Solidity code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# Medium Risk Findings (7)
## [[M-01] Extensive permissions for owner](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/4)
*Submitted by [Lambda](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/4)*

[contracts/RuniverseLandMinter.sol#L358](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/00a247e70de35d7a96d0ce03d66c0206b62e2f65/contracts/RuniverseLandMinter.sol#L358)<br>
[contracts/RuniverseLand.sol#L195](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/ea5fce62baeabf3f9d067ad747cee521d7be3a8a/contracts/RuniverseLand.sol#L195)<br>
[contracts/RuniverseLandMinter.sol#L513](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/00a247e70de35d7a96d0ce03d66c0206b62e2f65/contracts/RuniverseLandMinter.sol#L513)

Some privileged functions are often unavoidable in smart contracts. However, in these contracts, the privileges are (unnecessarily) very extensive and without checks on the smart contract side:

1.  He can use `ownerMint` (or `ownerMintUsingTokenId`) to mint an arbitrary number of tokens. While these functions should be used for private minting, there is nothing restricting the owner from minting more than 10,924 plots and using the function later on to mint additional plots.
2.  He can use `setVestingStart` / `setVestingEnd` / `setLastVestingGlobalId` to change the vesting configuration at any point. This can cause already vested tokens to suddenly become unvested (and therefore untransferable). Furthermore, there is no restriction on the parameters, so an owner could for instance set a vesting end that is 100 years in the future.
3.  He can change the plots that are available per size at any time with `setPlotsAvailablePerSize`. Therefore, a user might think that he buys a rare plot size, but the plot size becomes very common afterwards, destroying the value of his NFT.

Therefore, the user currently has to trust the owner that he does not perform any of the previously described actions.

### Recommended Mitigation Steps

1.  Only allow the owner to mint up to 10,924 plots and only allow it before the other phases have started (mintlist / claimlist / public sale).
2.  Remove these functions, these should be immutable parameters such that a user can be sure that his vesting date never changes.
3.  Remove this function, this should be immutable such that the rarities of NFTs cannot be changed arbitarily.

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/4#issuecomment-1373702786):**
 > After further talking with other Judges, I believe the finding to be valid and of Medium Severity.
> 
> While the fact that the admin has certain privileges is OOS.
> 
> The fact that:
> - Minting can go over the cap (unchecked minting / broken invariant)
> - Rarity could be changed indirectly (broken invariant)
> 
> Is worth noting.
> 
> For these reasons, am assigning a Medium Severity.

**[msclecram (Forgotten Runiverse) confirmed and commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/4#issuecomment-1378228355):**
 > We updated the code with the next changes:<br>
> - Vesting assert for starting time
> - OwnerMint assert to avoid minting more than plots available per size
> - We removed SetPlotsAvailablePerSize
> 
> https://github.com/bisonic-official/plot-contract/commit/ea8abd7faffde4218232e22ba5d8402e37d96878



***

## [[M-02] `ownerMintUsingTokenId` can brick the whole contract](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/6)
*Submitted by [Lambda](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/6)*

[contracts/RuniverseLandMinter.sol#L392](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/00a247e70de35d7a96d0ce03d66c0206b62e2f65/contracts/RuniverseLandMinter.sol#L392)

With the function `ownerMintUsingTokenId`, it is possible for the owner to mint a token with an arbitrary token ID. However, this can brick the whole contract and cause a situation where no more mints / buys are possible. This happens when a token ID is minted with that function that is later on also generated with `ownerGetNextTokenId`. In that case, the call to `runiverseLand.mintTokenId` will fail because the function calls `_mint` internally, which reverts when the token ID was already minted:

```solidity
function _mint(address to, uint256 tokenId) internal virtual {
        require(to != address(0), "ERC721: mint to the zero address");
        require(!_exists(tokenId), "ERC721: token already minted");
				...
}
```

Another problem with this function is that the owner can encode an arbitrary `plotSize` in this `tokenId` (e.g., also the ones with ID > 4 that are defined in `IRuniverseLand`, but are not for sale).

### Proof Of Concept

Let's say we have `plotsMinted[0] = 101` and `plotsMinted[1] = plotsMinted[2] = plotsMinted[3] = plotsMinted[4] = plotGlobalOffset = 0`. The owner uses the function to mint the token ID which corresponds to the token encoding `[102][102][0]` (112150186059264). He might not even be aware of that because it is not immediately visible in the decimal representation that 112150186059264 corresponds to `[102][102][0]`. This mint increases `plotsMinted[0]` to 102. When a user now tries to mint for plot size ID 0, the function `_mintTokens` calls `ownerGetNextTokenId(0)`, which will return `[102][102][0]` = 112150186059264. This will cause the mint to fail because this ID was already minted.

### Recommended Mitigation Steps

Remove the function `ownerMintUsingTokenId` or implement checks that the provided token ID wil not collide with a future token ID (by decoding it and checking that the provided `globalCounter` / `localCounter` are impossible).

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/6#issuecomment-1364270656):**
 > Specific inconsistent state caused by setters, per discussion with other judges I will flag and judge separately.
> 
> May end up grouping under admin privilege but will give it a chance vs a more generic report.

**[msclecram (Forgotten Runiverse) confirmed](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/6)**

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/6#issuecomment-1372119062):**
 > Per similar discussion to [`#10`](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/10) and [`#11`](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/11), the Warden has shown a way in which the function can cause an inconsistent state, which will cause reverts.
> 
> Because the codebase has checks to avoid inconsistent states, but this finding shows a way to sidestep them, I agree with Medium Severity.

**[msclecram (Forgotten Runiverse) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/6#issuecomment-1378228738):**
 > We updated the code with the next changes:<br>
> - We removed ownerMintUsingTokenId
> 
> https://github.com/bisonic-official/plot-contract/commit/ea8abd7faffde4218232e22ba5d8402e37d96878



***

## [[M-03] `setPlotsAvailablePerSize` does not work correctly](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/7)
*Submitted by [Lambda](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/7)*

[contracts/RuniverseLandMinter.sol#L513](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/00a247e70de35d7a96d0ce03d66c0206b62e2f65/contracts/RuniverseLandMinter.sol#L513)

The function `setPlotsAvailablePerSize` can be used for two things:

1.  Decreasing the number of plots that is available for a certain size
2.  Increase the number of plots that is available for a certain size

However, in both cases it can introduce errors that can brick parts of the contract. In case 1 where the number of plots is decreased, it is possible that the new number is smaller than `plotsMinted` for a specific ID. This will cause an underflow in `getAvailableLands`, which subtracts these values:

```solidity
        plotsAvailableBySize[0] = plotsAvailablePerSize[0] - plotsMinted[0];
        plotsAvailableBySize[1] = plotsAvailablePerSize[1] - plotsMinted[1];
        plotsAvailableBySize[2] = plotsAvailablePerSize[2] - plotsMinted[2];
        plotsAvailableBySize[3] = plotsAvailablePerSize[3] - plotsMinted[3];
        plotsAvailableBySize[4] = plotsAvailablePerSize[4] - plotsMinted[4];
```

On the other hand, when the number of available plots per size is increased, the minting can fail. This can happen because the `MAX_SUPPLY` in `RuniverseLand` is set to 70000, which is also the sum of all `plotsAvailablePerSize` entries. When the sum of these entries is increased beyond 70000, some tokens cannot be minted, because the `MAX_SUPPLY` is already reached.

### Proof Of Concept

`plotsAvailablePerSize[0]` is changed by the owner to 54500, all other entries are kept the same. Because more users prefer cheap plots, they buy all 54500 plots of size 8x8 and 15500 plots of size 16x16. However, this means that 70000 tokens are minted and no one can buy the 32x32 or 64x64, leading to a substantial financial loss for the protocol (because larger investors might have bought them later, but can no longer do so).

### Recommended Mitigation Steps

Enforce that

1.  All entries are larger than `plotsMinted`
2.  The new entries sum up to 70000 (or to a number that is <= 70000 if decreasing the max supply should be allowed)

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/7#issuecomment-1364289072):**
 > Lack of check to guarantee invariant.

**[msclecram (Forgotten Runiverse) acknowledged](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/7)**

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/7#issuecomment-1372066531):**
 > Similarly to [`#10`](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/10) and [`#11`](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/11), the Warden has shown a way for an invariant to be broken based on configuration.
> 
> Because certain aspects of the codebase are using the invariants which can be bypassed as shown above, I agree with Medium Severity.

**[msclecram (Forgotten Runiverse) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/7#issuecomment-1378229009):**
 > We updated the code with the next changes:<br>
> - We removed `setPlotsAvailablePerSize`
> 
> https://github.com/bisonic-official/plot-contract/commit/ea8abd7faffde4218232e22ba5d8402e37d96878



***

## [[M-04] `RuniverseLandMinter._mintTokensUsingTokenId` does not verify that the `tokenId` matches the corresponding `plotSize`](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/10)
*Submitted by [cccz](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/10), also found by [hansfriese](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/25)*

The first eight digits of the RuniverseLand TokenID indicate the corresponding plotSize of the NFT.<br>
Owner can call `RuniverseLandMinter.ownerMintUsingTokenId` directly to mint the NFT for a specific TokenID.<br>
In `RuniverseLandMinter._mintTokensUsingTokenId`, there is no verification that the first eight bits of the tokenId match the plotSize parameter, which allows the owner to bypass the plotsAvailablePerSize limit.

```solidity
    function _mintTokensUsingTokenId(
        IRuniverseLand.PlotSize plotSize,
        uint256 tokenId,
        address recipient
    ) private {
        uint256 numPlots = 1;
        require(
            plotsMinted[uint256(plotSize)] <
                plotsAvailablePerSize[uint256(plotSize)],
            "All plots of that size minted"
        );
        require(
            plotsMinted[uint256(plotSize)] + numPlots <=
                plotsAvailablePerSize[uint256(plotSize)],
            "Trying to mint too many plots"
        );

        plotsMinted[uint256(plotSize)] += 1;


        runiverseLand.mintTokenId(recipient, tokenId, plotSize);
    }
```

For example, the plotSize parameter provided by the owner when calling ownerMintUsingTokenId is 8 &ast; 8, while the plotSize contained in the tokenId is 128 &ast; 128, thus bypassing the plotsAvailablePerSize limit.

Also, once RuniverseLands with mismatched tokenId and plotSize are minted, the supply of RuniverseLands with different plotSize will no longer be correct because the plotsMinted variable is incorrectly calculated.

### Proof of Concept

[contracts/RuniverseLandMinter.sol#L362-L393](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLandMinter.sol#L362-L393)

### Recommended Mitigation Steps

Consider verifying in `RuniverseLandMinter._mintTokensUsingTokenId` that the first eight bits of the tokenId match the plotSize parameter.

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/10#issuecomment-1364282352):**
 > Additional possibly broken invariant due to Admin Privilege.

**[msclecram (Forgotten Runiverse) confirmed](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/10)**

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/10#issuecomment-1372064351):**
 > Similarly to [`#11`](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/11) the Warden has shown a way to bypass specific checks which offer an invariant, in this case the invariant is the fact that plotSizes are capped, which can be broken by using `ownerMintUsingTokenId` in an unintended way.
> 
> Because the lack of checks allows that, whereas the rest of the codebase offers checks to prevent that, I agree with Medium Severity.

**[msclecram (Forgotten Runiverse) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/10#issuecomment-1378229677):**
 > We updated the code with the next changes:<br>
> - We removed `_mintTokensUsingTokenId`
> 
> https://github.com/bisonic-official/plot-contract/commit/ea8abd7faffde4218232e22ba5d8402e37d96878



***

## [[M-05] `secondaryMinter` may break `plotsAvailablePerSize`](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/11)
*Submitted by [cccz](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/11)*

RuniverseLand allows primaryMinter and secondaryMinter to mint NFT.

        function mintTokenId(
            address recipient,
            uint256 tokenId,
            PlotSize size
        ) public override nonReentrant {
            require(numMinted < MAX_SUPPLY, "All land has been minted");
            require(
                _msgSender() == primaryMinter || _msgSender() == secondaryMinter,
                "Not a minter"
            );
            numMinted += 1;
            emit LandMinted(recipient, tokenId, size);    
            
            _mint(recipient, tokenId);
        }

RuniverseLandMinter, as one of them, will have a limit on the number of NFTs with different PlotSize

```solidity
    uint256[] public plotsAvailablePerSize = [
        52500, // 8x8
        16828, // 16x16
        560, // 32x32
        105, // 64x64
        7 // 128x128
    ];
```

This will be checked in `_mintTokens`

```solidity
    function _mintTokens(
        IRuniverseLand.PlotSize plotSize,
        uint256 numPlots,
        address recipient
    ) private {        
        require(
            plotsMinted[uint256(plotSize)] <
                plotsAvailablePerSize[uint256(plotSize)],
            "All plots of that size minted"
        );        
        require(
            plotsMinted[uint256(plotSize)] + numPlots <=
                plotsAvailablePerSize[uint256(plotSize)],
            "Trying to mint too many plots"
        );        
        for (uint256 i = 0; i < numPlots; i++) {

            uint256 tokenId = ownerGetNextTokenId(plotSize);            
            plotsMinted[uint256(plotSize)] += 1; 
```

But the other minter is not limited and can mint RuniverseLand with any tokenID, thus breaking the plotsAvailablePerSize limit.

### Proof of Concept

[contracts/RuniverseLand.sol#L88-L102](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLand.sol#L88-L102)<br>
[contracts/RuniverseLandMinter.sol#L323-L341](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLandMinter.sol#L323-L341)

### Recommended Mitigation Steps

Consider making RuniverseLandMinter the only minter for RuniverseLand.

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/11#issuecomment-1364280090):**
 > Risk of broken invariant, will flag, unsure about severity.

**[msclecram (Forgotten Runiverse) acknowledged](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/11)**

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/11#issuecomment-1372062498):**
 > The warden has shown a way to bypass specific checks, while the function is privileged, the lack of checks is inconsistent with the checks applied in other parts of the codebase.
> 
> For this reason, I agree with Medium Severity.



***

## [[M-06] `RuniverseLand.sol#mint()` can be bricked](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/16)
*Submitted by [Dravee](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/16), also found by [Lambda](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/26) and [hansfriese](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/21)*

[contracts/RuniverseLand.sol#L77](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLand.sol#L77)<br>
[contracts/RuniverseLand.sol#L101](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLand.sol#L101)

The `mint()` function uses `numMinted` to generate the `tokenId`:

```solidity
File: RuniverseLand.sol
72:     function mint(address recipient, PlotSize size)
73:         public
74:         override
75:         returns (uint256)
76:     {
77:         uint256 tokenId = numMinted;
78:         mintTokenId(recipient, tokenId, size);
79:         return tokenId;
80:     }
```

This `numMinted` value corresponds to the `totalSupply()`:

```solidity
File: RuniverseLand.sol
145:     function totalSupply() public view returns (uint256) {
146:         return numMinted;
147:     }
```

However, the `mintTokenId()` function can be called with any `tokenId`:

```solidity
File: RuniverseLand.sol
088:     function mintTokenId(
089:         address recipient,
090:         uint256 tokenId,
091:         PlotSize size
092:     ) public override nonReentrant {
093:         require(numMinted < MAX_SUPPLY, "All land has been minted");
094:         require(
095:             _msgSender() == primaryMinter || _msgSender() == secondaryMinter,
096:             "Not a minter"
097:         );
098:         numMinted += 1;
099:         emit LandMinted(recipient, tokenId, size);
100:         
101:         _mint(recipient, tokenId);
102:     }
```

Imagine the following scenario:

*   The contracts just got deployed and `numMinted == 0`
*   `primaryMinter` calls `mintTokenId()` with `tokenId == 1`
    *   Now `numMinted == 1`
*   `secondaryMinter` calls `mint()`
    *   In this case, `tokenId == numMinted`
    *   `_mint()` gets called with `tokenId == 1` which already exists, so it fails

### Recommended Mitigation Steps

Given that `RuniverseLand.sol#mint()` isn't called in `RuniverseLandMinter`, I feel like it should simply be deleted.<br>
`primaryMinter` or `secondaryMinter` can simply call `mintTokenId()` directly.

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/16#issuecomment-1364275657):**
 > Valid concern based on external requirements + the fact that arbitrary data can be inputted.

**[msclecram (Forgotten Runiverse) confirmed](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/16)**

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/16#issuecomment-1372060003):**
 > The warden has shown an inconsistency in using `mint` and `mintTokenId` which can cause the bricking of the minting functionality.
> 
> Because this is an undesirable scenario which can happen via ordinary operations, I agree with Medium Severity.

**[msclecram (Forgotten Runiverse) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/16#issuecomment-1378230053):**
 > We updated the code with the next changes:<br>
> - We removed mint method from interface and contract
> 
> https://github.com/bisonic-official/plot-contract/commit/ea8abd7faffde4218232e22ba5d8402e37d96878



***

## [[M-07] Grief on `transfers` due to `vestingStart` during vesting](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/17)
*Submitted by [Dravee](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/17), also found by [hansfriese](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/24)*

[contracts/ERC721Vestable.sol#L99-L101](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/ERC721Vestable.sol#L99-L101)<br>
[contracts/ERC721Vestable.sol#L44](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/ERC721Vestable.sol#L44)<br>
[contracts/ERC721Vestable.sol#L65](https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/ERC721Vestable.sol#L65)

*Past similar finding with the same severity: <https://github.com/code-423n4/2022-05-runes-findings/issues/30>*

While centralization risk is acknowledged by the team & the C4udit tool: this may lead to loss of functionality (grief).

### Proof of concept

There is no requirement for the start time to be before the end time:

```solidity
File: ERC721Vestable.sol
099:     function _setVestingStart(uint48 _newVestingStart) internal virtual { 
100:         vestingStart = _newVestingStart; //@audit-issue can be set after vesting end
101:     }
...
106:     function _setVestingEnd(uint48 _newVestingEnd) internal virtual {
107:         require(
108:             _newVestingEnd > vestingStart,
109:             "End must be greater than start"
110:         );
111:         vestingEnd = _newVestingEnd;
112:     }
```

Changing the start time in such a way (by error) can break the logic of transfer during the vesting period:

*   `ERC721Vestable.sol#_beforeTokenTransfer()`:

```solidity
File: ERC721Vestable.sol
31:     function _beforeTokenTransfer(
32:         address from,
33:         address to,
34:         uint256 tokenId
35:     ) internal virtual override {
36:         super._beforeTokenTransfer(from, to, tokenId);
37:         uint256 globalId = getGlobalId(tokenId);
38:         if (
39:             vestingEnabled &&
40:             from != address(0) && // minting
41:             globalId <= lastVestingGlobalId && 
42:             block.timestamp < vestingEnd 
43:         ) {
44:             uint256 vestingDuration = vestingEnd - vestingStart; //@audit Griefing if vestingStart > vestingEnd (possible)
45:             uint256 chunk = vestingDuration / lastVestingGlobalId;
46:             require(
47:                 block.timestamp >= (chunk * globalId) + vestingStart,
48:                 "Not vested"
49:             );
50:         }
51:         
52:     }
```

While less severe, it can also break the following view function:

*   `ERC721Vestable.sol#vestsAt()`:

```solidity
   65:         uint256 vestingDuration = vestingEnd - vestingStart;
```

### Recommended Mitigation Steps

I believe that `vestingStart` and `vestingEnd` should be immutable/constants. But in case the sponsor still wants them to be editable: consider adding the following check:

```diff
File: ERC721Vestable.sol
099:     function _setVestingStart(uint48 _newVestingStart) internal virtual { 
+ 100:         require(_newVestingStart < vestingEnd, "End must be greater than start");
100:         vestingStart = _newVestingStart; //@audit-issue can be set after vesting end
101:     }
```

The process to edit the vesting period would then be to first edit `vestingEnd` if the new `vestingStart` is going to be bigger that it.

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/17#issuecomment-1364295348):**
 > Worth flagging, unclear on severity and if the sponsor intended for vesting to be changeable or not.
>
 > With the info we have, we know this undefined state can be achieved, so probably valid, but acceptable nofix by sponsor.

**[msclecram (Forgotten Runiverse) confirmed](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/17)**

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/17#issuecomment-1371948068):**
 > In contrast to other admin findings this one:<br>
> - Shows an inconsistency in setters, from a convention that was already followed
> - Shows how this inconsistency can cause an issue with the contract, not merely a Admin Privilege
>
 > For the reasons above am judging the finding Medium Severity.

**[msclecram (Forgotten Runiverse) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/17#issuecomment-1378230463):**
 > We updated the code with the next changes:<br>
> - We added require for starting time
> 
> https://github.com/bisonic-official/plot-contract/commit/ea8abd7faffde4218232e22ba5d8402e37d96878



***

# Low Risk and Non-Critical Issues

For this audit, 4 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/3) by **Lambda** received the top score from the judge.

*The following wardens also submitted reports: [Dravee](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/23), [hansfriese](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/22), and [cccz](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/14).*

## [01] `mintTokenId` uses `_mint` instead of `_safeMint`
`RuniverseLand.mintTokenId` uses `_mint` instead of `_safeMint` to mint NFTs. Therefore, no callback will be performed for smart contracts and it is not checked if a smart contract can handle NFTs. This should not be a huge security problem, because this function is usually called when a user / contract explicitly requests that a token is minted (and pays for it). It would be very stupid (and a user error) to do this from a contract where the NFT is afterwards not retrievable. However, the private sale minting is a bit different because this is done by the owner to a list of given addresses. I do not know how this list is created, but there it could potentially happen that a user accidentally provided a smart contract (e.g., a smart contract wallet) and did not think about this. So you might want to consider using `_safeMint` instead.

## [02] Unnecessary check in `RuniverseLand.forwardERC20s`
The check `address(msg.sender) != address(0)` in `forwardERC20s` is not necessary because this function is only callble by the owner and `msg.sender` cannot be `address(0)` (or it could be in theory, but no one has the private key for `address(0)`, or we would have huge problems).

## [03] `Runiverse.withdrawAll` uses `send` instead of `call` to send ETH
The function `withdrawAll` uses `send` for sending ETH. This function only provides a 2300 gas stipend, which might not be sufficient. However, in this context this is not very severe in my opinion because the function is only used for recovery purposes and if the owner is currently a recipient that uses more than 2300 gas in its `receive` function, it could be temporarily changed to an EOA, so no funds are lost.

## [04] Unused constant in `RuniverseLand`
The constant string `R` in `RuniverseLand` is used nowhere and can be removed.

## [05] Rounding Error in `ERC721Vestable._beforeTokenTransfer`
Because of the calculation in `_beforeTokenTransfer` which first divides and then multiplies, it can happen that all tokens are fully vested 10924 seconds (~3 hours) before the configured end, which may be undesirable. Consider rounding up in the first division.

## [06] `setGlobalIdOffset` / `setLocalIdOffsets` callable during claimlist phase
The functions `setGlobalIdOffset` & `setLocalIdOffsets` include the check
```solidity
require(!mintlistStarted(), "Can't change during mint");
```
However, this means that they are potentially callable during the claimlist phase (namely when the claimlist begins before the mintlist, which is quite probable in my opinion). To ensure consecutive counters (and no accidental collisions), this should not be possible, so consider requiring that both the mintlist and the claimlist has not started. 

## [07] `IRuniverseLand.PlotSize` contains non-mintable sizes
The enum `PlotSize` als contains entries for 256x256, ..., 4096x4096 plots, although they are not mintable using these contracts. Consider removing them in order to not confuse developers that built on top of the project and use this interface for the integration.

## [08] `RuniverseLandMinter.mintlisted` has unnecessary argument `_leaf`
The argument `_leaf` is only used to compare it with the hashed `_who` argument. This check does not add any benefit and requires all clients (which will often be some off-chain client for this view function) to also calculate this hash unnecessarily.

## [09] Unnecessary check in `RuniverseLand._mintTokens`
The first check in `_mintTokens` (`plotsMinted[uint256(plotSize)] < plotsAvailablePerSize[uint256(plotSize)]`) is unnecessary and could be removed. The second check (`plotsMinted[uint256(plotSize)] + numPlots <= plotsAvailablePerSize[uint256(plotSize)]`) implies the first one, so there is no need to perform the first one (except for the different error message, but I do not think that this is worth the gas).

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/3#issuecomment-1364269540):**
 > **[01] mintTokenId uses `_mint` instead of `_safeMint`**<br>
> Low
> 
> **[02] Unnecessary check in `RuniverseLand.forwardERC20s`**<br>
> Refactoring
> 
> **[03] `Runiverse.withdrawAll` uses send instead of call to send ETH**<br>
> Low
> 
> **[04] Unused constant in RuniverseLand**<br>
> Refactoring
> 
> **[05] Rounding Error in `ERC721Vestable._beforeTokenTransfer`**<br>
> Low
> 
> **[06] `setGlobalIdOffset` / `setLocalIdOffsets` callable during claimlist phase**<br>
> Good catch.<br>
> Low
>  
> **[07] `IRuniverseLand.PlotSize` contains non-mintable sizes**<br>
> Refactoring
> 
> **[08] `RuniverseLandMinter.mintlisted` has unnecessary argument `_leaf`**<br>
> Refactoring
> 
> **[09] Unnecessary check in `RuniverseLand._mintTokens`**<br>
> Refactoring
>
 > 4 Low, 5 Refactoring

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/3#issuecomment-1373634826):**
 > Best Report because of strong impact of findings (Low Impact).



***

# Gas Optimizations

For this audit, 1 report was submitted detailing gas optimizations. The [report highlighted below](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/15) by **Dravee** received the top score from the judge.

## [G-01] Using `calldata` instead of `memory`

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

## [G-02] Redundant checks and storage readings

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

Indeed, here, the second check is logically equivalent to the 1st as `numPlots == 1`.<br>
It wouldn't make much sense either to say "Trying to mint too many plots" when the number of plots being minted is constantly 1.<br>
The message "All plots of that size minted" is the only one relevant.

Gas saved: **around 200**

## [G-03] Refactoring the logic of a for-loop to save gas

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

## [G-04] Multiple accesses of a mapping/array should use a local variable cache

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

## [G-05] Caching storage values in memory

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

## [G-06] (Proposal) Variables that should be constant/immutable

There are some variables that are very unlikely to change

- File: ERC721Vestable.sol

```solidity
16:     /// @notice the tokens from 0 to lastVestedTokenId will vest over time
17:     uint256 public lastVestingGlobalId = 10924;
18: 
19:     /// @notice the time the vesting started
20:     uint256 public vestingStart = 1671840000; // Dec 24th, 2022
21: 
22:     /// @notice the time the vesting ends
23:     uint256 public vestingEnd = 1734998400; // Dec 24th, 2024
```

- File: RuniverseLandMinter.sol (`vault` is actually the multisig `owner`)

```solidity
19:     /// @notice Address to the vault where we can withdraw
20:     address payable public vault; 
```

Consider deleting their setters and marking them as immutable to save a massive amount of gas (**20 000 gas per constant**)

## [G-07] Switching between `1` and `2` instead of `0` and `1` (or `false` and `true`) is more gas efficient

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

## [G-08] Make some variables smaller for storage packing and favorizing the user

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

## [G-09] Explicitly assigning a default value in storage wastes gas

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

## [G-10] `++numMinted` costs less gas compared to `numMinted += 1`

*This one isn't in a for-loop and isn't covered by [c4udit](https://gist.github.com/JustDravee/4b13fe17b1b8aa7e01a2369b92719e38#GAS-8)*

```diff
contracts/RuniverseLand.sol:
-   98:         numMinted += 1;
+   98:         ++numMinted; 
```

## [G-11] `++plotsMinted[uint256(plotSize)]` costs less gas compared to `plotsMinted[uint256(plotSize)] += 1`

*This one isn't covered by [c4udit](https://gist.github.com/JustDravee/4b13fe17b1b8aa7e01a2369b92719e38#GAS-8)*

```diff
File: RuniverseLandMinter.sol
- 341:             plotsMinted[uint256(plotSize)] += 1;          
+ 341:             ++plotsMinted[uint256(plotSize)];          
- 379:             plotsMinted[uint256(plotSize)] += 1;          
+ 379:             ++plotsMinted[uint256(plotSize)];          
```

## [G-12] Increments can be unchecked in for-loops

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

## [G-13] `require()` statements that check input arguments should be at the top of the function

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

## [G-14] `vault` cannot realistically be `address(0)`

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

## [G-15] Unnecessary check for `msg.sender` being `address(0)`

Unless there's a way to own `address(0)` (in which case, please DM), these checks should be deleted:

```solidity
contracts/RuniverseLand.sol:
  221:         require(address(msg.sender) != address(0), "req sender");

contracts/RuniverseLandMinter.sol:
  557:         require(address(msg.sender) != address(0), "req sender");
```

## [G-16] Unchecking arithmetics operations that can't underflow/overflow

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

## [G-17] Splitting `require()` statements that use `&&` saves gas

See [this issue](https://github.com/code-423n4/2022-01-xdefi-findings/issues/128) which describes the fact that there is a larger deployment gas cost, but with enough runtime calls, the change ends up being cheaper.

Affected code (saving around **3 gas** per instance):

```solidity
RuniverseLandMinter.sol:212:        require(numPlots > 0 && numPlots <= 20, "Mint from 1 to 20 plots");        
RuniverseLandMinter.sol:230:        require(numPlots > 0 && numPlots <= 20, "Mint from 1 to 20 plots");
```

## [G-18] Using private rather than public for constants saves gas

If needed, the value can be read from the verified contract source code. Savings are due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table.

```solidity
RuniverseLand.sol:40:    uint256 public constant MAX_SUPPLY = 70000;
RuniverseLand.sol:57:    string public constant R = "I should like to save the Shire, if I could"; 
```

## [G-19] Upgrade pragma

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

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/15#issuecomment-1364318384):**
 > First finding will save over 3k gas alone.
> 
> Rest of advice is also pretty good.

**[msclecram (Forgotten Runiverse) confirmed and commented](https://github.com/code-423n4/2022-12-forgotten-runiverse-findings/issues/15#issuecomment-1378235629):**
 > Commits with gas optimizations and C4 reports: https://github.com/bisonic-official/plot-contract/compare/21e6c8002555f0c8d91eec02abad3d85ec63cae2...main



***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 Audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
