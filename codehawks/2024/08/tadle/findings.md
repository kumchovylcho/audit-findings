# [H-01] abortOfferStatus will not be updated at PreMarkets.sol::listOffer() since the struct is loaded into memory instead of storage

## Summary

https://github.com/Cyfrin/2024-08-tadle/blob/04fd8634701697184a3f3a5558b41c109866e5f8/src/core/PreMarkets.sol#L342

`originOfferInfo.abortOfferStatus = AbortOfferStatus.SubOfferListed;` will not get updated since it is loaded from memory, instead of storage.

## Vulnerability Details

It's not updating the abortOfferStatus.

## Impact

When the problem gets fixed, it will open a new problem:

It could lead to DoS inside of `PreMarkets.sol::abortAskOffer()` when checking for the abortOfferStatus since the abortOfferStatus could also be `SubOfferListed`

```solidity
if (offerInfo.abortOfferStatus != AbortOfferStatus.Initialized) {
    revert InvalidAbortOfferStatus(AbortOfferStatus.Initialized, offerInfo.abortOfferStatus);
}
```

## Tools Used

Manual Review

## Recommendations

Change loading to storage instead of to memory.

```diff
/// @dev change abort offer status when offer settle type is turbo
if (makerInfo.offerSettleType == OfferSettleType.Turbo) {
    address originOffer = makerInfo.originOffer;
-  OfferInfo memory originOfferInfo = offerInfoMap[originOffer];
+ OfferInfo storage originOfferInfo = offerInfoMap[originOffer];

    if (_collateralRate != originOfferInfo.collateralRate) {
        revert InvalidCollateralRate();
    }

    originOfferInfo.abortOfferStatus = AbortOfferStatus.SubOfferListed;
}
Consider a different approach at abortAskOffer() for the abortOfferStatus check.
```
