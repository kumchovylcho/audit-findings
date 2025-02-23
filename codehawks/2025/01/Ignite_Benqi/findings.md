[Low-1] Wrong event param used in staking.sol::setSlippage()

## Summary

The emitted event will give wrong data, if the `_slippage` passed in the function param is below the `minSlippage` or above the `maxSlippage`.

## Vulnerability Details

When the `_slippage` parameter in the function is adjusted to fall within the bounds defined by `minSlippage` and `maxSlippage`, the second param of the event still uses `_slippage` instead of using `boundedSlippage`. It will emit wrong data only when the provided `_slippage` is out of bounds.

https://github.com/Cyfrin/2025-01-benqi/blob/f24a5550694e5ff24b059334feeb387b3576ffc9/zeeve/contracts/staking.sol#L327

## Impact

Off chain applications and Dapps relie on informations given by events, this could lead to several problems in applications, misleading the end user.

## Tools Used

Manual Review

## Recommendations

```diff
    function setSlippage(
        uint256 _slippage
    ) external onlyRole(BENQI_ADMIN_ROLE) {
        uint256 boundedSlippage = _slippage;
        if (_slippage < minSlippage) {
            boundedSlippage = minSlippage;
        } else if (_slippage > maxSlippage) {
            boundedSlippage = maxSlippage;
        }
        uint256 oldSlippage = slippage;
        slippage = boundedSlippage;
-       emit SlippageUpdated(oldSlippage, _slippage);
+       emit SlippageUpdated(oldSlippage, boundedSlippage);
    }
```
