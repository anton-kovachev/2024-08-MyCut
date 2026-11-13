# Incorrect Bonus Distribution Calculation Locks Funds Permanently

**Root + Impact**

## Description

* According to the MyCut protocol specification, after 90 days the consolemanager should "take a cut of the remaining pool and the remainder is distributed equally to those who claimed in time." 
* However, the `closePot()` function incorrectly calculates the bonus distribution by dividing the bonus pool by the total number of players instead of the number of actual claimants.

This causes underpayment to eligible claimants and permanently locks the difference in the contract.

```solidity
function closePot() external onlyOwner {
    if (block.timestamp - i_deployedAt < 90 days) {
        revert Pot__StillOpenForClaim();
    }
    if (remainingRewards > 0) {
        uint256 managerCut = remainingRewards / managerCutPercent;
        i_token.transfer(msg.sender, managerCut);

        //@audit: Divides by i_players.length (total players)
@>      uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
        
        //@audit: But distributes to claimants.length (only those who claimed)
@>      for (uint256 i = 0; i < claimants.length; i++) {
            _transferReward(claimants[i], claimantCut);
        }
    }
}
```

The issue arises when:
- `i_players.length` = total number of players (e.g., 3)
- `claimants.length` = number who claimed (e.g., 2)

The calculation divides the bonus pool by 3, but only distributes to 2 claimants. This leaves `bonusPool * (1 - claimants.length/i_players.length)` permanently locked in the contract.

## Risk

**Likelihood:**

High - This bug triggers whenever fewer than all players claim their rewards, which is the primary use case the protocol is designed for (distributing unclaimed rewards).

**Impact:**

High - Multiple severe consequences:

1. **Permanent Fund Locking**: Portion of bonus pool remains locked forever with no recovery mechanism
2. **Claimants Underpaid**: Users who claimed in time receive less than their fair share
3. **Violates Core Business Logic**: Contradicts the fundamental promise that the remainder is "distributed equally to those who claimed"
4. **Scaling Impact**: More unclaimed rewards → more locked funds (worse with fewer claimants)

## Proof of Concept

The test `testUnclaimedRewardDistribution_incorrect_result` demonstrates this vulnerability:

```solidity
function testUnclaimedRewardDistribution_incorrect_result()
    public
    mintAndApproveTokens
{
    vm.startPrank(user);
    address player3 = makeAddr("player3");

    address[] memory new_players = new address[](3);
    new_players[0] = player1;
    new_players[1] = player2;
    new_players[2] = player3;

    rewards = [500, 500, 1000];
    totalRewards = 2000;
    contest = ContestManager(conMan).createContest(
        new_players,
        rewards,
        IERC20(ERC20Mock(weth)),
        totalRewards
    );
    ContestManager(conMan).fundContest(0);
    vm.stopPrank();

    // Only player1 and player2 claim (player3 doesn't claim 1000)
    vm.startPrank(player1);
    Pot(contest).claimCut();  // Claims 500
    vm.stopPrank();

    vm.startPrank(player2);
    Pot(contest).claimCut();  // Claims 500
    vm.stopPrank();

    // Remaining: 1000 (player3's unclaimed reward)
    
    vm.warp(91 days);
    
    uint256 userBalanceBefore = ERC20Mock(weth).balanceOf(user);
    uint256 claimantBalanceBefore_Player1 = ERC20Mock(weth).balanceOf(player1);
    uint256 claimantBalanceBefore_Player2 = ERC20Mock(weth).balanceOf(player2);

    vm.startPrank(user);
    ContestManager(conMan).closeContest(contest);
    vm.stopPrank();

    uint256 userBalanceAfter = ERC20Mock(weth).balanceOf(user);
    uint256 claimantBalanceAfter_Player1 = ERC20Mock(weth).balanceOf(player1);
    uint256 claimantBalanceAfter_Player2 = ERC20Mock(weth).balanceOf(player2);

    uint256 contestBalance = ERC20Mock(weth).balanceOf(contest);
    
    // EXPECTED (correct) behavior:
    // Remaining: 1000
    // Manager cut: 1000 / 10 = 100
    // Bonus pool: 1000 - 100 = 900
    // Claimants: 2 (player1 and player2)
    // Each claimant: 900 / 2 = 450
    
    assertEq(userBalanceAfter - userBalanceBefore, 100);  // Manager gets 100
    assertEq(claimantBalanceAfter_Player1 - claimantBalanceBefore_Player1, 450);  // Player1 gets 450
    assertEq(claimantBalanceAfter_Player2 - claimantBalanceBefore_Player2, 450);  // Player2 gets 450
    assertEq(contestBalance, 0);  // No funds left locked
}
```

**What Actually Happens (Buggy Code):**

```
Remaining rewards: 1000
Manager cut: 1000 / 10 = 100
Bonus pool: 1000 - 100 = 900

Calculation: 900 / i_players.length = 900 / 3 = 300 per claimant (WRONG!)
Distribution: 2 claimants × 300 = 600 distributed
Locked: 900 - 600 = 300 (33% of bonus pool PERMANENTLY LOCKED!)

Manager receives: 100 ✓
Player1 receives: 300 (should be 450) ✗
Player2 receives: 300 (should be 450) ✗
Locked in contract: 300 ✗
```

**Mathematical Impact:**

The amount locked is: `bonusPool × (i_players.length - claimants.length) / i_players.length`

Scenarios:
- 5 players, 2 claim: `bonusPool × 3/5 = 60%` locked
- 10 players, 3 claim: `bonusPool × 7/10 = 70%` locked
- 100 players, 10 claim: `bonusPool × 90/100 = 90%` locked

## Real-World Impact Example

```solidity
// Large contest scenario
Players: 50
Total rewards: 50,000 tokens
5 players claim: 10,000 tokens
45 players don't claim: 40,000 tokens remain

After 90 days:
- Remaining: 40,000
- Manager cut: 40,000 / 10 = 4,000
- Bonus pool: 36,000

BUGGY calculation:
- Each claimant cut: 36,000 / 50 = 720
- Distributed: 720 × 5 = 3,600
- LOCKED: 32,400 tokens (90% of bonus pool!)

CORRECT calculation:
- Each claimant cut: 36,000 / 5 = 7,200
- Distributed: 7,200 × 5 = 36,000
- Locked: 0
```

## Recommended Mitigation

Change the division to use the actual number of claimants instead of total players:

```diff
 function closePot() external onlyOwner {
     if (block.timestamp - i_deployedAt < 90 days) {
         revert Pot__StillOpenForClaim();
     }
     if (remainingRewards > 0) {
         uint256 managerCut = remainingRewards / managerCutPercent;
         i_token.transfer(msg.sender, managerCut);

-        uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
+        uint256 claimantCut = (remainingRewards - managerCut) / claimants.length;
         for (uint256 i = 0; i < claimants.length; i++) {
             _transferReward(claimants[i], claimantCut);
         }
     }
 }
```

**Additional Safety Check:**

Add a check to prevent division by zero if no one claimed:

```diff
 function closePot() external onlyOwner {
     if (block.timestamp - i_deployedAt < 90 days) {
         revert Pot__StillOpenForClaim();
     }
     if (remainingRewards > 0) {
         uint256 managerCut = remainingRewards / managerCutPercent;
         i_token.transfer(msg.sender, managerCut);

-        uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
-        for (uint256 i = 0; i < claimants.length; i++) {
-            _transferReward(claimants[i], claimantCut);
-        }
+        // Only distribute if there are claimants
+        if (claimants.length > 0) {
+            uint256 claimantCut = (remainingRewards - managerCut) / claimants.length;
+            for (uint256 i = 0; i < claimants.length; i++) {
+                _transferReward(claimants[i], claimantCut);
+            }
+        }
+        // If no claimants, remaining funds stay in contract
+        // Consider adding mechanism to allow manager to claim these
     }
 }
```

This fix ensures that the bonus pool is correctly distributed according to the protocol specification: "the remainder is distributed equally to those who claimed in time."
