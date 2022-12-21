## `mintTokenId` uses `_mint` instead of `_safeMint`
`RuniverseLand.mintTokenId` uses `_mint` instead of `_safeMint` to mint NFTs. Therefore, no callback will be performed for smart contracts and it is not checked if a smart contract can handle NFTs. This should not be a huge security problem, because this function is usually called when a user / contract explicitly requests that a token is minted (and pays for it). It would be very stupid (and a user error) to do this from a contract where the NFT is afterwards not retrievable. However, the private sale minting is a bit different because this is done by the owner to a list of given addresses. I do not know how this list is created, but there it could potentially happen that a user accidentally provided a smart contract (e.g., a smart contract wallet) and did not think about this. So you might want to consider using `_safeMint` instead.

## `MAX_SUPPLY` of `RuniverseLand` could be reached before `RuniverseLandMinter` mints all tokens because of secondary minter
`RuniverseLand` has a `MAX_SUPPLY` of 70000, which is also the sum of all `plotsAvailablePerSize` within `RuniverseLandMinter`. However, because there is also an (arbitrary) `secondaryMinter` in `RuniverseLand`, it can happen that this secondary minter mints some tokens. Then, `RuniverseLandMinter` will not be able to mint / sell some tokens because the limit was already reached.

## Unnecessary check in `RuniverseLand.forwardERC20s`
The check `address(msg.sender) != address(0)` in `forwardERC20s` is not necessary because this function is only callble by the owner and `msg.sender` cannot be `address(0)` (or it could be in theory, but no one has the private key for `address(0)`, or we would have huge problems).

## `Runiverse.withdrawAll` uses `send` instead of `call` to send ETH
The function `withdrawAll` uses `send` for sending ETH. This function only provides a 2300 gas stipend, which might not be sufficient. However, in this context this is not very severe in my opinion because the function is only used for recovery purposes and if the owner is currently a recipient that uses more than 2300 gas in its `receive` function, it could be temporarily changed to an EOA, so no funds are lost.

## `Runiverse.forwardERC20s` does not check the `transfer` return value
Because `Runiverse.forwardERC20s` does not check the return value of `transfer`, some ERC20 transfers that fail (when the token itself does not revert) will not cause the whole transaction to fail. Because this is only a recovery function, this is not very bad, but for usability / to recognize failed transfers (e.g., in a blockchain explorer), reverting in these cases might still be sensible.

## Unused constant in `RuniverseLand`
The constant string `R` in `RuniverseLand` is used nowhere and can be removed.

## Rounding Error in `ERC721Vestable._beforeTokenTransfer`
Because of the calculation in `_beforeTokenTransfer` which first divides and then multiplies, it can happen that all tokens are fully vested 10924 seconds (~3 hours) before the configured end, which may be undesirable. Consider rounding up in the first division.

## setGlobalIdOffset / setLocalIdOffsets callable during claimlist phase
The functions `setGlobalIdOffset` & `setLocalIdOffsets` include the check
```solidity
require(!mintlistStarted(), "Can't change during mint");
```
However, this means that they are potentially callable during the claimlist phase (namely when the claimlist begins before the mintlist, which is quite probable in my opinion). To ensure consecutive counters (and no accidental collisions), this should not be possible, so consider requiring that both the mintlist and the claimlist has not started. 

## `IRuniverseLand.PlotSize` contains non-mintable sizes
The enum `PlotSize` als contains entries for 256x256, ..., 4096x4096 plots, although they are not mintable using these contracts. Consider removing them in order to not confuse developers that built on top of the project and use this interface for the integration.

## `RuniverseLandMinter.mintlisted` has unnecessary argument `_leaf`
The argument `_leaf` is only used to compare it with the hashed `_who` argument. This check does not add any benefit and requires all clients (which will often be some off-chain client for this view function) to also calculate this hash unnecessarily.

## Unnecessary check in `RuniverseLand._mintTokens`
The first check in `_mintTokens` (`plotsMinted[uint256(plotSize)] < plotsAvailablePerSize[uint256(plotSize)]`) is unnecessary and could be removed. The second check (`plotsMinted[uint256(plotSize)] + numPlots <= plotsAvailablePerSize[uint256(plotSize)]`) implies the first one, so there is no need to perform the first one (except for the different error message, but I do not think that this is worth the gas).