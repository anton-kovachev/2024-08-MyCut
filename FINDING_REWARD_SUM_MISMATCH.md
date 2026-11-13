# Missing Reward Sum Validation Causes Fund Locking or Insolvency

**Root + Impact**

## Description

The MyCut protocol allows the owner to create contests where players can claim their allocated rewards. However, the `createContest()` function in ContestManager.sol accepts an arbitrary `totalRewards` parameter without validating it against the actual sum of the `rewards[]` array. This unvalidated value is then stored and used to fund the Pot contract, creating a critical accounting mismatch.

```solidity
function createContest(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards)
    public
    onlyOwner
    returns (address)
{
    //@audit: totalRewards is NOT validated against sum(rewards[])
    // Create a new Pot contract
    Pot pot = new Pot(players, rewards, token, totalRewards);
    contests.push(address(pot));
@>  contestToTotalRewards[address(pot)] = totalRewards; // Unvalidated value stored
    return address(pot);
}

function fundContest(uint256 index) public onlyOwner {
    Pot pot = Pot(contests[index]);
    IERC20 token = pot.getToken();
@>  uint256 totalRewards = contestToTotalRewards[address(pot)]; // Uses unvalidated value

    if (token.balanceOf(msg.sender) < totalRewards) {
        revert ContestManager__InsufficientFunds();
    }

    //@audit: Transfers unvalidated amount that may not match actual reward allocations
@>  token.transferFrom(msg.sender, address(pot), totalRewards);
}
```

In the Pot constructor, this mismatch becomes critical:

```solidity
constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) {
    i_players = players;
    i_rewards = rewards;
    i_token = token;
    i_totalRewards = totalRewards;
@>  remainingRewards = totalRewards; // Set to unvalidated totalRewards
    i_deployedAt = block.timestamp;

    for (uint256 i = 0; i < i_players.length; i++) {
@>      playersToRewards[i_players[i]] = i_rewards[i]; // Actual allocations from rewards[]
    }
}
```

The contract uses `totalRewards` for funding and accounting (`remainingRewards`), but uses the individual `rewards[]` values for player allocations. When these don't match, the protocol breaks in two ways:

1. **Underfunding** (`totalRewards < sum(rewards[])`): Insufficient tokens in Pot to honor all claims
2. **Overfunding** (`totalRewards > sum(rewards[])`): Excess tokens permanently locked in Pot

## Risk

**Likelihood:**

Medium-High - While the owner is trusted, this is an easy configuration mistake to make when:
- Manually calculating reward totals
- Updating reward values without updating totalRewards
- Copy-pasting from previous contest configurations
- No automated tooling validates the inputs

**Impact:**

High - Both scenarios cause severe consequences:

**Underfunding Scenario:**
- Later claimants cannot receive their rewards (transfer fails or insufficient balance)
- Protocol appears functional initially but fails when funds depleted
- Players lose their rightfully allocated rewards
- Reputation damage from failed claims

**Overfunding Scenario:**
- Excess tokens permanently locked in Pot contract
- No mechanism to recover excess funds after all claims completed
- Wasted capital that could fund other contests
- After 90 days, `closePot()` redistributes based on incorrect `remainingRewards` value

## Proof of Concept

**Scenario 1: Underfunding**

```solidity
function testUnderfundedContest() public {
    address[] memory players = new address[](3);
    players[0] = alice;
    players[1] = bob;
    players[2] = charlie;
    
    uint256[] memory rewards = new uint256[](3);
    rewards[0] = 100e18;  // 100 tokens
    rewards[1] = 200e18;  // 200 tokens
    rewards[2] = 300e18;  // 300 tokens
    // Actual sum = 600 tokens
    
    // Owner mistakenly passes 400 instead of 600
    uint256 incorrectTotal = 400e18;
    
    vm.startPrank(owner);
    address contest = ContestManager(conMan).createContest(
        players,
        rewards,
        IERC20(token),
        incorrectTotal  // Only 400 tokens instead of 600!
    );
    
    // Fund with the incorrect amount
    ContestManager(conMan).fundContest(0);
    vm.stopPrank();
    
    // Alice claims successfully (100 tokens)
    vm.prank(alice);
    Pot(contest).claimCut();  // ✓ Success
    
    // Bob claims successfully (200 tokens)
    vm.prank(bob);
    Pot(contest).claimCut();  // ✓ Success
    
    // Charlie tries to claim but only 100 tokens remain
    vm.prank(charlie);
    vm.expectRevert(); // Transfer fails - insufficient balance
    Pot(contest).claimCut();  // ✗ FAIL - Charlie allocated 300 but only 100 left
}
```

**Scenario 2: Overfunding with Locked Funds**

```solidity
function testOverfundedContest() public {
    address[] memory players = new address[](2);
    players[0] = alice;
    players[1] = bob;
    
    uint256[] memory rewards = new uint256[](2);
    rewards[0] = 100e18;  // 100 tokens
    rewards[1] = 200e18;  // 200 tokens
    // Actual sum = 300 tokens
    
    // Owner mistakenly passes 500 instead of 300
    uint256 incorrectTotal = 500e18;
    
    vm.startPrank(owner);
    address contest = ContestManager(conMan).createContest(
        players,
        rewards,
        IERC20(token),
        incorrectTotal  // 500 tokens instead of 300!
    );
    
    // Fund with the excessive amount
    ContestManager(conMan).fundContest(0);
    vm.stopPrank();
    
    uint256 potBalanceBefore = token.balanceOf(contest);
    assertEq(potBalanceBefore, 500e18);  // 500 tokens in Pot
    
    // Alice claims 100
    vm.prank(alice);
    Pot(contest).claimCut();
    
    // Bob claims 200
    vm.prank(bob);
    Pot(contest).claimCut();
    
    // 200 tokens remain locked with no legitimate claimants
    uint256 potBalanceAfter = token.balanceOf(contest);
    assertEq(potBalanceAfter, 200e18);  // 200 tokens PERMANENTLY LOCKED
    
    // Even after 90 days, closePot() can't properly handle this
    vm.warp(block.timestamp + 91 days);
    vm.prank(owner);
    Pot(contest).closePot();
    
    // The 200 remaining tokens get incorrectly distributed
    // based on flawed remainingRewards accounting
}
```

**Scenario 3: Real-World Calculation Error**

```solidity
// Owner creates contest with rewards in different decimals
// Token A (6 decimals) vs Token B (18 decimals)
uint256[] memory rewards = new uint256[](3);
rewards[0] = 100e6;   // 100 tokens (6 decimals)
rewards[1] = 200e6;   // 200 tokens (6 decimals)
rewards[2] = 300e6;   // 300 tokens (6 decimals)
// Sum = 600e6

// Owner calculates in wrong decimals
uint256 totalRewards = 600e18;  // Used 18 decimals by mistake!

// Result: Massive overfunding (600e18 vs 600e6)
// Or underfunding if reversed
```

## Recommended Mitigation

**Option 1: Validate in ContestManager (Recommended)**

Add validation before creating the Pot to catch errors early:

```diff
+error ContestManager__RewardSumMismatch();

 function createContest(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards)
     public
     onlyOwner
     returns (address)
 {
+    // Validate totalRewards matches sum of rewards array
+    uint256 rewardSum = 0;
+    for (uint256 i = 0; i < rewards.length; i++) {
+        rewardSum += rewards[i];
+    }
+    if (rewardSum != totalRewards) {
+        revert ContestManager__RewardSumMismatch();
+    }
+
     // Create a new Pot contract
     Pot pot = new Pot(players, rewards, token, totalRewards);
     contests.push(address(pot));
     contestToTotalRewards[address(pot)] = totalRewards;
     return address(pot);
 }
```

**Option 2: Calculate in ContestManager (Most Robust)**

Remove the `totalRewards` parameter entirely and calculate it automatically:

```diff
-function createContest(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards)
+function createContest(address[] memory players, uint256[] memory rewards, IERC20 token)
     public
     onlyOwner
     returns (address)
 {
+    // Calculate totalRewards automatically
+    uint256 totalRewards = 0;
+    for (uint256 i = 0; i < rewards.length; i++) {
+        totalRewards += rewards[i];
+    }
+    
     // Create a new Pot contract
     Pot pot = new Pot(players, rewards, token, totalRewards);
     contests.push(address(pot));
     contestToTotalRewards[address(pot)] = totalRewards;
     return address(pot);
 }
```

**Option 3: Validate in Pot Constructor (Defense in Depth)**

Add validation at the contract boundary:

```diff
+error Pot__RewardSumMismatch();

 constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) {
+    // Validate totalRewards matches sum of rewards
+    uint256 rewardSum = 0;
+    for (uint256 i = 0; i < players.length; i++) {
+        rewardSum += rewards[i];
+    }
+    if (rewardSum != totalRewards) {
+        revert Pot__RewardSumMismatch();
+    }
+
     i_players = players;
     i_rewards = rewards;
     i_token = token;
     i_totalRewards = totalRewards;
     remainingRewards = totalRewards;
     i_deployedAt = block.timestamp;

     for (uint256 i = 0; i < i_players.length; i++) {
         playersToRewards[i_players[i]] = i_rewards[i];
     }
 }
```

**Recommended Approach:** Implement **Option 2** (automatic calculation) in ContestManager for the cleanest API, combined with **Option 3** (validation) in Pot constructor for defense in depth. This eliminates the possibility of human error while maintaining contract-level safety checks.
