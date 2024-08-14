# [L-01] Removing attester inside RegistryFactory.sol won't work for duplicate addresses

[Link to finding](https://github.com/Cyfrin/2024-07-biconomy/blob/9590f25cd63f7ad2c54feb618036984774f3879d/contracts/factory/RegistryFactory.sol#L61)

# Summary

Calling the `removeAttester()` function inside of `RegistryFactory.sol` won't remove duplicate attesters.

# Vulnerability Details

Since `addAttester()` does not have a mechanism to check if the address already exists and the only way to remove an attester is from `removeAttester()` removing them one by one will cost more resources than if you could remove them all at once.

# Impact

You won't be able to remove duplicate attesters and you will need to make several transactions costing you more money.

Tools Used
Manual Review

# Recommendations

Looping backwards because that way the array won't get re-indexed and we won't skip an attester.

```diff
function removeAttester(address attester) external onlyOwner {
+        for (uint256 i = attesters.length; i > 0; i--) {
-        for (uint256 i = 0; i < attesters.length; i++) {
+            uint256 index = i - 1;
+            if (attesters[index] == attester) {
-            if (attesters[i] == attester) {
+                attesters[index] = attesters[attesters.length - 1];
-                attesters[i] = attesters[attesters.length - 1];
                attesters.pop();
-                break;
        }
    }
}
```
