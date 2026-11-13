# Security Audit: MyCut Protocol - August 2024

## Executive Summary

This security audit of the MyCut protocol (Cyfrin/2024-08-MyCut) identified **4 critical vulnerabilities** that collectively compromise the core functionality and security of the reward distribution system. These findings range from permanent fund locking to accounting mismatches and failed transaction handling.

**Severity Breakdown:**
- **Critical:** 4 findings
- **Total Impact:** Up to 90% of funds can be permanently locked with zero recovery mechanism

---

## Audit Findings

### 1. 🔴 Missing Duplicate Player Validation Causes Reward Loss and Accounting Corruption

**Severity:** CRITICAL  
**Status:** Unpatched

#### Root Cause
The `Pot` constructor accepts a `players[]` array without validating for duplicate addresses. When the same address appears multiple times, each occurrence overwrites the previous mapping entry, permanently losing earlier reward allocations.

#### Vulnerable Code
```solidity
constructor(address[] memory players, uint256[] memory rewards, ...) {
    // No duplicate check
    for (uint256 i = 0; i < i_players.length; i++) {
        playersToRewards[i_players[i]] = i_rewards[i];  // ⚠️ Overwrites on duplicate
    }
}
```

#### Impact
- **Direct Loss:** Player loses all reward allocations except the last one
- **Accounting Broken:** `totalRewards` in Pot doesn't match claimable rewards
- **Bonus Distribution Error:** `closePot()` uses incorrect `i_players.length` for calculation
- **Example:** Contest with players `[Alice, Bob, Alice]` and rewards `[1000, 500, 200]` results in Alice only being able to claim 200 (loses 1000) and 1000 tokens permanently locked

#### Likelihood
**Medium-High** - Can occur through:
- Owner making configuration mistakes
- Copy-paste errors in player arrays
- Off-chain tooling bugs generating duplicates

#### Recommended Fix
```solidity
// Option 1: Validate in Pot constructor
mapping(address => bool) private playerExists;

constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) {
    for (uint256 i = 0; i < players.length; i++) {
        if (playerExists[players[i]]) {
            revert Pot__DuplicatePlayer(players[i]);
        }
        playerExists[players[i]] = true;
        playersToRewards[players[i]] = rewards[i];
    }
    // ... rest of constructor
}

// Option 2: Also validate in ContestManager.createContest()
for (uint256 i = 0; i < players.length; i++) {
    for (uint256 j = i + 1; j < players.length; j++) {
        if (players[i] == players[j]) {
            revert ContestManager__DuplicatePlayer(players[i]);
        }
    }
}
```

---

### 2. 🔴 Incorrect Bonus Distribution Calculation Locks Funds Permanently

**Severity:** CRITICAL  
**Status:** Unpatched

#### Root Cause
The `closePot()` function divides the bonus pool by `i_players.length` (total players) instead of `claimants.length` (actual claimants), leaving funds permanently locked.

#### Vulnerable Code
```solidity
function closePot() external onlyOwner {
    if (remainingRewards > 0) {
        uint256 managerCut = remainingRewards / managerCutPercent;
        i_token.transfer(msg.sender, managerCut);

        uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;  // ❌ WRONG
        for (uint256 i = 0; i < claimants.length; i++) {  // ❌ Only distributes to claimants
            _transferReward(claimants[i], claimantCut);
        }
    }
}
```

#### Impact
- **Fund Locking:** `bonusPool × (i_players.length - claimants.length) / i_players.length` permanently locked
- **Underpayment:** Claimants receive less than their fair share
- **Scaling Risk:** More unclaimed rewards = more locked funds
- **Real Example:** 
  - 50 players, 5 claim (90% don't claim)
  - Remaining: 40,000 tokens
  - Expected bonus per claimant: 7,200 tokens
  - Actual bonus per claimant: 720 tokens
  - **Locked forever: 32,400 tokens (90% of bonus!)**

#### Likelihood
**High** - Triggers whenever not all players claim, which is the intended use case

#### Recommended Fix
```solidity
function closePot() external onlyOwner {
    if (block.timestamp - i_deployedAt < 90 days) {
        revert Pot__StillOpenForClaim();
    }
    if (remainingRewards > 0) {
        uint256 managerCut = remainingRewards / managerCutPercent;
        i_token.transfer(msg.sender, managerCut);

        // ✅ FIX: Use claimants.length instead of i_players.length
        if (claimants.length > 0) {
            uint256 claimantCut = (remainingRewards - managerCut) / claimants.length;
            for (uint256 i = 0; i < claimants.length; i++) {
                _transferReward(claimants[i], claimantCut);
            }
        }
    }
}
```

---

### 3. 🔴 Missing Reward Sum Validation Causes Fund Locking or Insolvency

**Severity:** CRITICAL  
**Status:** Unpatched

#### Root Cause
The `createContest()` function accepts an arbitrary `totalRewards` parameter without validating it against `sum(rewards[])`. This creates two failure scenarios:

1. **Underfunding:** `totalRewards < sum(rewards[])` → Later claimants can't claim
2. **Overfunding:** `totalRewards > sum(rewards[])` → Excess locked forever

#### Vulnerable Code
```solidity
function createContest(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) 
    public onlyOwner returns (address)
{
    // ❌ No validation that totalRewards == sum(rewards)
    Pot pot = new Pot(players, rewards, token, totalRewards);
    contests.push(address(pot));
    contestToTotalRewards[address(pot)] = totalRewards;  // Unvalidated value stored
    return address(pot);
}

function fundContest(uint256 index) public onlyOwner {
    // Uses unvalidated totalRewards
    uint256 totalRewards = contestToTotalRewards[address(pot)];
    token.transferFrom(msg.sender, address(pot), totalRewards);
}
```

#### Impact

**Underfunding Scenario:**
```solidity
Rewards: [100, 200, 300]  // Sum = 600
totalRewards: 400  // ⚠️ WRONG!

Alice claims 100 ✓
Bob claims 200 ✓
Charlie tries to claim 300 ✗ (only 100 left, transfer fails or reverts)
```

**Overfunding Scenario:**
```solidity
Rewards: [100, 200]  // Sum = 300
totalRewards: 500  // ⚠️ WRONG!

Alice claims 100 ✓
Bob claims 200 ✓
200 tokens PERMANENTLY LOCKED (no legitimate claimant)
```

#### Likelihood
**Medium-High** - Easy configuration mistake:
- Manual reward total calculation errors
- Copy-paste from old configurations
- Decimal place confusion (6 vs 18 decimals)

#### Recommended Fix
```solidity
// ✅ Option 1: Validate in createContest()
error ContestManager__RewardSumMismatch();

function createContest(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards)
    public onlyOwner returns (address)
{
    // Calculate and validate sum
    uint256 rewardSum = 0;
    for (uint256 i = 0; i < rewards.length; i++) {
        rewardSum += rewards[i];
    }
    if (rewardSum != totalRewards) {
        revert ContestManager__RewardSumMismatch();
    }
    
    Pot pot = new Pot(players, rewards, token, totalRewards);
    contests.push(address(pot));
    contestToTotalRewards[address(pot)] = totalRewards;
    return address(pot);
}

// ✅ Option 2: Automatically calculate totalRewards (removes parameter)
function createContest(address[] memory players, uint256[] memory rewards, IERC20 token)
    public onlyOwner returns (address)
{
    uint256 totalRewards = 0;
    for (uint256 i = 0; i < rewards.length; i++) {
        totalRewards += rewards[i];
    }
    
    Pot pot = new Pot(players, rewards, token, totalRewards);
    contests.push(address(pot));
    contestToTotalRewards[address(pot)] = totalRewards;
    return address(pot);
}

// ✅ Option 3: Validate in Pot constructor (defense in depth)
constructor(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards) {
    uint256 rewardSum = 0;
    for (uint256 i = 0; i < players.length; i++) {
        rewardSum += rewards[i];
    }
    if (rewardSum != totalRewards) {
        revert Pot__RewardSumMismatch();
    }
    // ... rest of constructor
}
```

**Recommendation:** Implement Option 2 for the cleanest API, combined with Option 3 for defense in depth.

---

### 4. 🔴 Unchecked Token Transfer Return Value Allows State Updates on Failed Claims

**Severity:** CRITICAL  
**Status:** Unpatched

#### Root Cause
The `claimCut()` and `closePot()` functions don't validate the return value of ERC20 `transfer()` calls. Many tokens (ZRX, EURS, etc.) return `false` on failure instead of reverting.

#### Vulnerable Code
```solidity
function claimCut() public {
    address player = msg.sender;
    uint256 reward = playersToRewards[player];
    if (reward <= 0) {
        revert Pot__RewardNotFound();
    }
    
    // State changes BEFORE transfer validation
    playersToRewards[player] = 0;
    remainingRewards -= reward;
    claimants.push(player);
    _transferReward(player, reward);  // ❌ Return value ignored
}

function _transferReward(address player, uint256 reward) internal {
    i_token.transfer(player, reward);  // ❌ No return value check
}
```

#### Impact
When transfer returns `false` without reverting:
1. **Permanent Reward Loss:** Player's `playersToRewards[player]` is zeroed, preventing re-claim
2. **Accounting Mismatch:** `remainingRewards` decremented despite no tokens sent
3. **Unfair Distribution:** Failed claimants added to `claimants[]`, receiving bonus in `closePot()` despite never getting initial reward
4. **Manager Overpayment:** Manager cut calculated from inflated `remainingRewards`

#### Example Attack/Failure Scenario
```
1. Pot created with 1000 tokens
2. Alice calls claimCut() expecting 300 tokens
3. Transfer returns false (due to blacklist, rate limit, or token bug)
4. Alice's reward set to 0 (state updated)
5. remainingRewards decremented to 700
6. Alice added to claimants array
7. Alice receives 0 tokens but can never re-claim
8. After 90 days, Alice receives bonus as if she claimed successfully
```

#### Likelihood
**High** - Affects any non-standard ERC20 that returns false:
- ZRX token
- EURS token
- Many older token implementations
- Insufficient Pot balance

#### Recommended Fix
```solidity
// ✅ Option 1: Use SafeERC20 (Recommended - Industry Standard)
import {SafeERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/utils/SafeERC20.sol";

contract Pot is Ownable(msg.sender) {
    using SafeERC20 for IERC20;
    
    function _transferReward(address player, uint256 reward) internal {
        i_token.safeTransfer(player, reward);  // ✅ Handles both reverting and non-reverting tokens
    }
    
    function closePot() external onlyOwner {
        if (block.timestamp - i_deployedAt < 90 days) {
            revert Pot__StillOpenForClaim();
        }
        if (remainingRewards > 0) {
            uint256 managerCut = remainingRewards / managerCutPercent;
            i_token.safeTransfer(msg.sender, managerCut);  // ✅ Safe transfer
            
            if (claimants.length > 0) {
                uint256 claimantCut = (remainingRewards - managerCut) / claimants.length;
                for (uint256 i = 0; i < claimants.length; i++) {
                    _transferReward(claimants[i], claimantCut);
                }
            }
        }
    }
}

// ✅ Option 2: Check return value manually
function _transferReward(address player, uint256 reward) internal {
    bool success = i_token.transfer(player, reward);
    if (!success) {
        revert Pot__TransferFailed();
    }
}

// ✅ Option 3: Combine with CEI pattern
function claimCut() public {
    address player = msg.sender;
    uint256 reward = playersToRewards[player];
    
    // Checks
    if (reward <= 0) {
        revert Pot__RewardNotFound();
    }
    
    // Effects (state changes)
    playersToRewards[player] = 0;
    remainingRewards -= reward;
    claimants.push(player);
    
    // Interactions (external calls last)
    i_token.safeTransfer(player, reward);  // ✅ With SafeERC20
}
```

**Recommendation:** Use SafeERC20 for maximum compatibility and industry best practices.

---

## Vulnerability Interaction Analysis

These vulnerabilities compound each other, creating cascading failures:

### Scenario: All 4 Vulnerabilities Triggered

```
Setup:
- Players: [Alice, Bob, Alice]  (Finding #1: Duplicate!)
- Rewards: [1000, 500, 200]
- totalRewards: 1700 (Finding #3: Should be 1700 but now only 700 claimable)
- Token: ZRX (non-reverting on failure)

Execution:
1. Alice tries to claim (Gets 200, loses 1000 to duplicate)
2. Bob claims successfully (Gets 500)
3. Transfer to Alice returns false (Finding #4)
4. Alice's reward zeroed anyway
5. After 90 days, closePot():
   - Remaining: 1000 tokens (but accounting shows 1200 or wrong amount)
   - Manager cut: 100 tokens
   - Bonus pool: 900 tokens
   - Calculation: 900 / 3 = 300 per person (Finding #2)
   - Actual claimants: 2 → 300 × 2 = 600 distributed
   - LOCKED: 300 tokens + prior issues = 1000+ tokens permanent loss (58%!)

Total Impact:
✗ Alice lost 1000 tokens (duplicate)
✗ 300 tokens locked in contract (wrong divisor)
✗ Failed transfer not caught
✗ Reward sum mismatch compounds problem
```

---

## Modified Files in This Audit

### Current Changes (Formatting Only - No Security Fixes)
1. **src/Pot.sol**
   - Added console import (for logging)
   - Code formatting improvements
   - No vulnerability fixes applied

2. **src/ContestManager.sol**
   - Code formatting improvements
   - No vulnerability fixes applied

3. **test/TestMyCut.t.sol**
   - Test updates for vulnerability demonstration

### New Documentation Files
- **FINDING_DUPLICATE_PLAYER.md** - Detailed analysis of Finding #1
- **FINDING_INCORRECT_BONUS_DISTRIBUTION.md** - Detailed analysis of Finding #2
- **FINDING_REWARD_SUM_MISMATCH.md** - Detailed analysis of Finding #3
- **FINDING_UNCHECKED_TRANSFER.md** - Detailed analysis of Finding #4

---

## Recommended Remediation Priority

### Phase 1: Immediate (Before Any Contest Deployment)
1. ✅ Add duplicate player validation (Finding #1)
2. ✅ Add reward sum validation (Finding #3)
3. ✅ Implement SafeERC20 for transfers (Finding #4)

### Phase 2: Core Logic Fix
4. ✅ Fix bonus distribution calculation (Finding #2)

### Phase 3: Hardening (Defense in Depth)
- Add multiple layers of validation
- Implement comprehensive event logging
- Add emergency pause mechanism
- Create contest finalization validation function

---

## Testing Recommendations

All findings include proof-of-concept tests in test files:
- `testUnclaimedRewardDistribution_duplicate_player_no_check_wrong_reward()`
- `testUnclaimedRewardDistribution_incorrect_result()`
- `testRewardSumValidation()`
- `testClaimCutWithFailedTokenTransfer_Does_Not_Revert()`

**Recommendation:** Use these tests as regression tests after implementing fixes.

---

## Compliance and Standards

This audit was conducted against:
- ERC20 standard specifications
- OpenZeppelin security best practices
- Common Solidity vulnerability patterns (CWE-191, CWE-682, CWE-252)
- Checks-Effects-Interactions pattern

---

## Conclusion

The MyCut protocol contains 4 critical vulnerabilities that must be addressed before production deployment. Collectively, these issues can result in up to 90% of contest funds being permanently locked with zero recovery mechanism.

**Status:** ⚠️ **NOT PRODUCTION READY**

**Next Steps:**
1. Implement recommended fixes for all 4 findings
2. Add comprehensive validation throughout
3. Run full test suite with POC tests
4. Consider formal verification for critical paths
5. Get re-audit after implementing fixes

---

## Audit Metadata

- **Repository:** Cyfrin/2024-08-MyCut
- **Audit Date:** August 2024
- **Auditor:** Security Audit Team
- **Findings:** 4 Critical
- **Files Analyzed:** src/Pot.sol, src/ContestManager.sol, test/TestMyCut.t.sol
- **Time Commitment:** Comprehensive analysis with POC tests
