### [L-01] `ERC721Vestable._setVestingStart()` should check `_newVestingStart<vestingEnd`.

- https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/ERC721Vestable.sol#L100

```solidity
    function _setVestingStart(uint256 _newVestingStart) internal virtual {
        //@audit
        require(vestingEnd > _newVestingStart, "End must be greater than start");
        vestingStart = _newVestingStart;
    }
```

### [L-02] `RuniverseLandMinter.ownerMintUsingTokenId()` doesn't check if `tokenId` and `plotSize` are matched.

- https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLandMinter.sol#L387-L393

```solidity
    function ownerMintUsingTokenId(
        IRuniverseLand.PlotSize plotSize,
        uint256 tokenId,
        address recipient
    ) public onlyOwner {
        _mintTokensUsingTokenId(plotSize, tokenId, recipient);
    }
```

When admin adds a custom `tokenId`, it doesn't check if the id and `plotSize` are matched. If they aren't matched, `plotsMinted` will be counted wrongly.

Even if it's an admin function, it would be good to validate for safety.

### [N-01] Typo

- https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLandMinter.sol#L124

"@return getPlotPrices uint256 plot type" => "@return getTokenIdPlotType uint256 plot type"

- https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/ERC721Vestable.sol#L54

"besting property" => "vesting property"

### [N-02] Missing checks for `address(0)`

- https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLandMinter.sol#L469
- https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLandMinter.sol#L480

The new values should be checked to prevent zero address.

### [N-03] Time values are not checked

- https://github.com/code-423n4/2022-12-forgotten-runiverse/blob/dcad1802bf258bf294900a08a03ca0d26d2304f4/contracts/RuniverseLandMinter.sol#L48

```solidity
    uint256 public publicMintStartTime = type(uint256).max;
    uint256 public mintlistStartTime = type(uint256).max;
    uint256 public claimsStartTime = type(uint256).max;
```

The above time values are used to decide the current minting phase.
Obviously these are supposed to be in specific orders, e.g. `claimsStartTime<mintlistStartTime<publicMintStartTime`.
But it is not checked on the setter functions.

```solidity
    function setPublicMintStartTime(uint256 _newPublicMintStartTime)
        public
        onlyOwner
    {
        publicMintStartTime = _newPublicMintStartTime;
    }

    /**
     * @dev Assigns a new mintlist start minting time.
     * @param _newAllowlistMintStartTime uint256 echo time in seconds.
     */
    function setMintlistStartTime(uint256 _newAllowlistMintStartTime)
        public
        onlyOwner
    {
        mintlistStartTime = _newAllowlistMintStartTime;
    }

    /**
     * @dev Assigns a new claimlist start minting time.
     * @param _newClaimsStartTime uint256 echo time in seconds.
     */
    function setClaimsStartTime(uint256 _newClaimsStartTime) public onlyOwner {
        claimsStartTime = _newClaimsStartTime;
    }
```
