# Missing Duplicate Player Validation Causes Reward Loss and Accounting Corruption

**Root + Impact**

## Description

According to the MyCut protocol specification, each player should be able to claim their allocated reward. However, the Pot constructor does not validate for duplicate player addresses in the `players` array, allowing the same address to appear multiple times. This causes the player's reward allocation to be silently overwritten in the mapping, resulting in permanent loss of earlier reward allocations and corrupted contract accounting.

The vulnerability occurs during the constructor's initialization loop:

```solidity
constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) {
    i_players = players;
    i_rewards = rewards;
    i_token = token;
    i_totalRewards = totalRewards;
    remainingRewards = totalRewards;
    i_deployedAt = block.timestamp;

    //@audit: No check for duplicate addresses
    for (uint256 i = 0; i < i_players.length; i++) {
        //@audit: If a player appears multiple times, their mapping value gets overwritten
        //@audit: Last entry wins, previous reward allocations are lost
@>      playersToRewards[i_players[i]] = i_rewards[i];
    }
}
```

When a duplicate address exists:
1. First occurrence sets `playersToRewards[player] = firstReward`
2. Second occurrence **overwrites** it: `playersToRewards[player] = secondReward`
3. The first reward becomes **permanently unclaimed** but remains in accounting
4. Player can only claim the last assigned value, **losing all previous allocations**

## Risk

**Likelihood:**

Medium - This can occur through:
- Owner making an honest mistake when creating a contest
- Copy-paste errors when preparing player arrays
- Off-chain tooling bugs that generate duplicate entries
- No validation prevents this during contest creation

**Impact:**

High - Multiple severe consequences:

1. **Permanent Reward Loss**: Players lose all but their last reward allocation with no recovery mechanism
2. **Broken Accounting**: `totalRewards` doesn't match sum of claimable rewards
   - Contract funded for 1700 but only 700 is claimable
   - 1000 tokens permanently locked
3. **Incorrect Bonus Distribution**: `closePot()` uses `i_players.length` (3) for calculation when actual unique players is fewer (2)
4. **Unfair Distribution**: Duplicate player counted multiple times in bonus pool division
5. **Silent Failure**: No error or event indicates the overwrite occurred

## Proof of Concept

The test `testUnclaimedRewardDistribution_duplicate_player_no_check_wrong_reward` demonstrates this vulnerability:

```solidity
function testUnclaimedRewardDistribution_duplicate_player_no_check_wrong_reward()
    public
    mintAndApproveTokens
{
    vm.startPrank(user);
    
    // Create players array with player1 appearing TWICE
    address[] memory new_players = new address[](3);
    new_players[0] = player1;  // First entry
    new_players[1] = player2;
    new_players[2] = player1;  // Duplicate entry!

    uint256 player1Reward = 1000;
    uint256 player2Reward = 500;
    uint256 player3Reward = 200;  // This will overwrite player1's reward

    rewards = [player1Reward, player2Reward, player3Reward];
    totalRewards = 1700;  // 1000 + 500 + 200
    
    contest = ContestManager(conMan).createContest(
        new_players,
        rewards,
        IERC20(ERC20Mock(weth)),
        totalRewards
    );
    ContestManager(conMan).fundContest(0);
    vm.stopPrank();

    // Constructor execution breakdown:
    // Loop iteration 0: playersToRewards[player1] = 1000
    // Loop iteration 1: playersToRewards[player2] = 500
    // Loop iteration 2: playersToRewards[player1] = 200  ← OVERWRITES!
    
    // Player1 tries to claim
    vm.startPrank(player1);
    Pot(contest).claimCut();
    vm.stopPrank();
    
    // Player2 claims
    vm.startPrank(player2);
    Pot(contest).claimCut();
    vm.stopPrank();

    vm.warp(91 days);
    uint256 claimantBalanceBefore_Player1 = ERC20Mock(weth).balanceOf(player1);
    
    // Player1 only received 200 (the last value), lost 1000!
    assert(claimantBalanceBefore_Player1 == player3Reward);  // 200
}
```

**What Happened:**

```
Setup:
- players[0] = player1, rewards[0] = 1000
- players[1] = player2, rewards[1] = 500  
- players[2] = player1, rewards[2] = 200 (duplicate!)
- totalRewards = 1700

Constructor Execution:
- playersToRewards[player1] = 1000  ✓
- playersToRewards[player2] = 500   ✓
- playersToRewards[player1] = 200   ✗ (overwrites 1000!)

Claiming:
- player1 calls claimCut(): receives 200 (LOST 1000!)
- player2 calls claimCut(): receives 500 ✓
- Total claimed: 700
- Locked forever: 1000

Contract State:
- Funded amount: 1700
- Claimable amount: 700 (200 + 500)
- Permanently locked: 1000 (58.8% of total funds!)
- i_players.length: 3 (but only 2 unique players)
```

## Detailed Impact Analysis

### Scenario 1: Duplicate Player - Small Contest

```solidity
Players: [Alice, Bob, Alice]
Rewards: [1000, 500, 200]
Total: 1700

Result:
- Alice mapping: 200 (lost 1000)
- Bob mapping: 500
- Alice claims: 200
- Bob claims: 500
- Locked: 1000 (58.8%)
```


### Additional Impact on closePot()

After 90 days when `closePot()` is called:

```solidity
// Remaining: 1000 (from Scenario 1)
uint256 managerCut = 1000 / 10 = 100;
// Bonus pool: 900

// WRONG calculation (uses total players, not unique)
uint256 claimantCut = 900 / i_players.length = 900 / 3 = 300;

// But only 2 claimants in array
// Distributes: 300 × 2 = 600
// Still locked: 300

// Final locked amount: 1000 - 100 - 600 = 300 (additional locked!)
```

**Compounding Issue**: The duplicate player problem **combines** with the incorrect bonus distribution, creating multiple layers of locked funds.

## Recommended Mitigation

Add duplicate player validation in the constructor:

```diff
+error Pot__DuplicatePlayer(address player);

 constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) {
+    // Validate no duplicate players
+    for (uint256 i = 0; i < players.length; i++) {
+        // Check if this player was already assigned a reward
+        if (playersToRewards[players[i]] != 0) {
+            revert Pot__DuplicatePlayer(players[i]);
+        }
+        // Assign reward
+        playersToRewards[players[i]] = rewards[i];
+    }
+
     i_players = players;
     i_rewards = rewards;
     i_token = token;
     i_totalRewards = totalRewards;
     remainingRewards = totalRewards;
     i_deployedAt = block.timestamp;
-
-    for (uint256 i = 0; i < i_players.length; i++) {
-        playersToRewards[i_players[i]] = i_rewards[i];
-    }
 }
```

**Note on Edge Case**: This mitigation assumes rewards are always non-zero. If zero rewards are valid, use a separate tracking mechanism:

```diff
+mapping(address => bool) private playerExists;
+error Pot__DuplicatePlayer(address player);

 constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) {
+    for (uint256 i = 0; i < players.length; i++) {
+        if (playerExists[players[i]]) {
+            revert Pot__DuplicatePlayer(players[i]);
+        }
+        playerExists[players[i]] = true;
+        playersToRewards[players[i]] = rewards[i];
+    }
+
     i_players = players;
     i_rewards = rewards;
     i_token = token;
     i_totalRewards = totalRewards;
     remainingRewards = totalRewards;
     i_deployedAt = block.timestamp;
-
-    for (uint256 i = 0; i < i_players.length; i++) {
-        playersToRewards[i_players[i]] = i_rewards[i];
-    }
 }
```

**Additional Recommendation**: Also add this validation in `ContestManager.createContest()` to fail fast:

```diff
+error ContestManager__DuplicatePlayer(address player);

 function createContest(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards)
     public
     onlyOwner
     returns (address)
 {
+    // Validate no duplicates before deploying Pot
+    for (uint256 i = 0; i < players.length; i++) {
+        for (uint256 j = i + 1; j < players.length; j++) {
+            if (players[i] == players[j]) {
+                revert ContestManager__DuplicatePlayer(players[i]);
+            }
+        }
+    }
+
     // Create a new Pot contract
     Pot pot = new Pot(players, rewards, token, totalRewards);
     contests.push(address(pot));
     contestToTotalRewards[address(pot)] = totalRewards;
     return address(pot);
 }
```

This prevents invalid contests from being created and wasting gas on deploying a broken Pot contract.
