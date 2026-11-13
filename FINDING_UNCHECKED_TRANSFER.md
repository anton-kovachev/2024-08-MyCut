# Unchecked Token Transfer Return Value Allows State Updates on Failed Claims

**Root + Impact**

## Description

According to the MyCut protocol specification, users should be able to "claim their cut of a Pot" within 90 days, after which the manager takes a cut and remaining funds are distributed to those who successfully claimed in time.

The `claimCut()` function fails to check the return value of the ERC20 `transfer()` call, allowing critical state updates even when the token transfer fails. This breaks the core business requirement that users can claim their rewards.

```solidity
function claimCut() public {
    address player = msg.sender;
    uint256 reward = playersToRewards[player];
    if (reward <= 0) {
        revert Pot__RewardNotFound();
    }
    //@audit: State changes happen before transfer validation
    playersToRewards[player] = 0;
    remainingRewards -= reward;
    claimants.push(player);
    _transferReward(player, reward);
}

function _transferReward(address player, uint256 reward) internal {
    //@audit: Return value not checked - transfer can return false without reverting
@>  i_token.transfer(player, reward);
}
```

Many ERC20 tokens return `false` on failed transfers instead of reverting (e.g., ZRX, EURS). The function updates all critical state variables (`playersToRewards`, `remainingRewards`, `claimants`) before the transfer occurs, and never validates the transfer succeeded.

## Risk

**Likelihood:**

High - This vulnerability affects any ERC20 token that returns false on failure rather than reverting. The issue is triggered on every `claimCut()` call when:
- The Pot contract has insufficient token balance
- The token implementation returns false for any reason
- Transfer restrictions or blacklists prevent the transfer

**Impact:**

High - Multiple severe consequences occur:
1. **Permanent Fund Loss**: Users lose their rewards permanently as their `playersToRewards[player]` is set to 0, preventing re-claims
2. **Accounting Mismatch**: `remainingRewards` decrements despite failed transfers, causing contract state to diverge from actual token balance
3. **Unfair Distribution**: Failed claimants are added to `claimants[]` array, making them eligible for bonus distribution in `closePot()` despite never receiving their initial reward
4. **Manager Receives Inflated Cut**: When `closePot()` executes, the manager's cut is calculated from inflated `remainingRewards` that doesn't match actual tokens held

## Proof of Concept

The test `testClaimCutWithFailedTokenTransfer_Does_Not_Revert` demonstrates this vulnerability:

```solidity
function testClaimCutWithFailedTokenTransfer_Does_Not_Revert()
    public
    mintAndApproveTokens
{
    vm.startPrank(user);
    contest = ContestManager(conMan).createContest(
        players,
        rewards,
        IERC20(ERC20Mock(weth)),
        4
    );
    ContestManager(conMan).fundContest(0);
    vm.stopPrank();
    
    // Player balance before claim
    uint256 balanceBefore = ERC20Mock(weth).balanceOf(player1);
    
    vm.startPrank(player1);
    // Mock the transfer to return false (failed transfer)
    vm.mockCall(
        address(Pot(contest).getToken()),
        abi.encodeWithSelector(
            IERC20.transfer.selector,
            player1,
            rewards[0]
        ),
        abi.encode(false)
    );
    
    // Claim does not revert despite failed transfer
    Pot(contest).claimCut();
    vm.stopPrank();
    
    // Player balance unchanged - NO TOKENS RECEIVED
    uint256 balanceAfter = ERC20Mock(weth).balanceOf(player1);
    uint256 playerRemainingReward = Pot(contest).checkCut(player1);
    
    // State was updated despite failed transfer
    assertEq(playerRemainingReward, 0);  // Reward set to 0
    assertEq(balanceAfter, balanceBefore); // No tokens received
}
```

**Attack Scenario:**
1. Pot is created with insufficient funding or uses a token that can return false
2. User calls `claimCut()`
3. Transfer returns `false` but function doesn't revert
4. User's reward is zeroed out and they're added to claimants
5. User permanently loses their reward
6. After 90 days, `closePot()` distributes remaining funds to claimants including the failed claimer
7. User who never received initial reward gets bonus distribution while actual funds remain locked

## Recommended Mitigation

**Option 1: Use SafeERC20 (Recommended)**

Replace direct `transfer()` calls with OpenZeppelin's `SafeERC20` library which handles both reverting and non-reverting tokens:

```diff
 // SPDX-License-Identifier: MIT
 pragma solidity ^0.8.20;
 
 import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
+import {SafeERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/utils/SafeERC20.sol";
 import {Ownable} from "lib/openzeppelin-contracts/contracts/access/Ownable.sol";
 
 contract Pot is Ownable(msg.sender) {
+    using SafeERC20 for IERC20;
+
     error Pot__RewardNotFound();
     error Pot__InsufficientFunds();
     error Pot__StillOpenForClaim();
```

```diff
 function _transferReward(address player, uint256 reward) internal {
-    i_token.transfer(player, reward);
+    i_token.safeTransfer(player, reward);
 }
```

```diff
 function closePot() external onlyOwner {
     if (block.timestamp - i_deployedAt < 90 days) {
         revert Pot__StillOpenForClaim();
     }
     if (remainingRewards > 0) {
         uint256 managerCut = remainingRewards / managerCutPercent;
-        i_token.transfer(msg.sender, managerCut);
+        i_token.safeTransfer(msg.sender, managerCut);
 
         uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
         for (uint256 i = 0; i < claimants.length; i++) {
             _transferReward(claimants[i], claimantCut);
         }
     }
 }
```

**Option 2: Check Return Value Manually**

```diff
 function _transferReward(address player, uint256 reward) internal {
-    i_token.transfer(player, reward);
+    bool success = i_token.transfer(player, reward);
+    if (!success) revert Pot__TransferFailed();
 }
```

**Option 3: Follow CEI Pattern + Use SafeERC20**

For defense in depth, also reorder `claimCut()` to follow Checks-Effects-Interactions:

```solidity
function claimCut() public {
    address player = msg.sender;
    uint256 reward = playersToRewards[player];
    
    // Checks
    if (reward <= 0) {
        revert Pot__RewardNotFound();
    }
    
    // Effects
    playersToRewards[player] = 0;
    remainingRewards -= reward;
    claimants.push(player);
    
    // Interactions (with SafeERC20)
    i_token.safeTransfer(player, reward);
}
```

**Recommended:** Use Option 1 (SafeERC20) as it handles both reverting and non-reverting ERC20 implementations correctly and is the industry standard for safe token transfers.
