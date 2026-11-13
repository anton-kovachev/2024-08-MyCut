# First Flight #23: MyCut

### Prize Pool

- High - 100 XP
- Medium - 20 XP
- Low - 2 XP

- Starts: August, 29 2024 Noon UTC

- Ends: September 05, 2024 Noon UTC

- nSLOC: 106

[//]: # (contest-details-open)

## About the Project

MyCut is a contest rewards distribution protocol which allows the set up and management of multiple rewards distributions, allowing authorized claimants 90 days to claim before the manager takes a cut of the remaining pool and the remainder is distributed equally to those who claimed in time!

### Actors

- Owner/Admin (Trusted) - Is able to create new Pots, close old Pots when the claim period has elapsed and fund Pots
- User/Player - Can claim their cut of a Pot

[//]: # (contest-details-close)

---

## 🔴 SECURITY AUDIT - August 2024

### ⚠️ CRITICAL FINDINGS IDENTIFIED

This repository contains a comprehensive security audit of the MyCut protocol. **4 CRITICAL vulnerabilities** have been identified that collectively compromise the core functionality and security of the reward distribution system.

**STATUS:** ❌ **NOT PRODUCTION READY** - All vulnerabilities must be remediated before mainnet deployment.

### Audit Summary

| Finding | Severity | Impact | Status |
|---------|----------|--------|--------|
| #1: Duplicate Player Validation | 🔴 CRITICAL | 100% reward loss for duplicates | Unpatched |
| #2: Bonus Distribution Calculation | 🔴 CRITICAL | 60-90% bonus pool locked | Unpatched |
| #3: Reward Sum Validation | 🔴 CRITICAL | Contract insolvency or overfunding | Unpatched |
| #4: Unchecked Transfer Return | 🔴 CRITICAL | State corruption, reward loss | Unpatched |

### Quick Links to Audit Documentation

- **📄 [AUDIT_README.md](./AUDIT_README.md)** - Executive summary and complete audit report
- **📄 [FINDING_DUPLICATE_PLAYER.md](./FINDING_DUPLICATE_PLAYER.md)** - Finding #1: Detailed analysis with POC and fixes
- **📄 [FINDING_INCORRECT_BONUS_DISTRIBUTION.md](./FINDING_INCORRECT_BONUS_DISTRIBUTION.md)** - Finding #2: Detailed analysis with POC and fixes
- **📄 [FINDING_REWARD_SUM_MISMATCH.md](./FINDING_REWARD_SUM_MISMATCH.md)** - Finding #3: Detailed analysis with POC and fixes
- **📄 [FINDING_UNCHECKED_TRANSFER.md](./FINDING_UNCHECKED_TRANSFER.md)** - Finding #4: Detailed analysis with POC and fixes

### Critical Findings Overview

#### 🔴 Finding #1: Missing Duplicate Player Validation

**Root Cause:** The `Pot` constructor doesn't validate for duplicate addresses in the players array.

**Impact:**
- Players can be added multiple times, causing reward mappings to be overwritten
- Permanent loss of earlier reward allocations
- Accounting mismatch between `totalRewards` and claimable amounts
- Up to 100% of duplicate player rewards lost

**Example:**
```solidity
Players: [Alice, Bob, Alice]
Rewards: [1000, 500, 200]

Result:
- Alice only claims 200 (last value)
- Lost: 1000 tokens (permanently locked)
```

**Recommended Fix:** Add duplicate validation in constructor or use a mapping to track existing players.

---

#### 🔴 Finding #2: Incorrect Bonus Distribution Calculation

**Root Cause:** The `closePot()` function divides the bonus pool by `i_players.length` instead of `claimants.length`.

**Impact:**
- Funds permanently locked in contract
- Claimants underpaid
- 60-90% of bonus pool can be locked depending on claim rate
- Violates core protocol specification

**Example:**
```
50 players, only 5 claim:
- Remaining: 40,000 tokens
- Expected per claimant: 7,200 tokens
- Actual per claimant: 720 tokens
- LOCKED: 32,400 tokens (90% of bonus!)
```

**Recommended Fix:** Change division to use `claimants.length` instead of `i_players.length`.

---

#### 🔴 Finding #3: Missing Reward Sum Validation

**Root Cause:** `createContest()` accepts arbitrary `totalRewards` without validating against `sum(rewards[])`.

**Impact:**
- **Underfunding:** Later claimants cannot receive their rewards
- **Overfunding:** Excess tokens permanently locked
- Contract insolvency or wasted capital
- Easy configuration mistake for contract owner

**Example - Underfunding:**
```solidity
Rewards: [100, 200, 300]  // Sum = 600
totalRewards: 400  // ⚠️ WRONG!

Result: Charlie cannot claim 300 (transfer fails)
```

**Example - Overfunding:**
```solidity
Rewards: [100, 200]  // Sum = 300
totalRewards: 500  // ⚠️ WRONG!

Result: 200 tokens permanently locked
```

**Recommended Fix:** Either validate or automatically calculate totalRewards from the rewards array.

---

#### 🔴 Finding #4: Unchecked Token Transfer Return Value

**Root Cause:** `transfer()` return value not checked. Many ERC20 tokens (ZRX, EURS) return `false` instead of reverting on failure.

**Impact:**
- State updated despite failed transfer
- Permanent reward loss (user's mapping zeroed)
- Player added to claimants array without receiving tokens
- Unfair bonus distribution after 90 days
- Manager receives inflated cut

**Example Attack:**
```
1. User calls claimCut()
2. Transfer returns false (non-reverting token)
3. User's reward set to 0
4. User added to claimants array
5. User never receives initial reward
6. User still gets bonus after 90 days (unfair!)
```

**Recommended Fix:** Use SafeERC20 or check return value manually.

---

### Vulnerability Interaction Analysis

These vulnerabilities compound each other. A scenario with all 4 triggered:

```
Setup: Players [Alice, Bob, Alice], Rewards [1000, 500, 200]
Findings triggered:
  #1: Duplicate player (Alice appears twice)
  #3: totalRewards mismatch
  #4: Failed transfer (non-reverting token)
  #2: Wrong bonus distribution

Result: 1000+ tokens (58%+ of total) permanently locked
```

### Remediation Priority

**Phase 1 (Immediate):**
1. Add duplicate player validation
2. Add reward sum validation
3. Implement SafeERC20 for transfers

**Phase 2 (Core Logic):**
4. Fix bonus distribution calculation

**Phase 3 (Hardening):**
- Add multiple validation layers
- Implement comprehensive event logging
- Add emergency pause mechanism
- Create contract finalization validation

---

### Testing

All findings include proof-of-concept tests in `test/TestMyCut.t.sol`:
- `testUnclaimedRewardDistribution_duplicate_player_no_check_wrong_reward()`
- `testUnclaimedRewardDistribution_incorrect_result()`
- `testRewardSumValidation()`
- `testClaimCutWithFailedTokenTransfer_Does_Not_Revert()`

### Production Readiness

| Aspect | Status | Notes |
|--------|--------|-------|
| Security | ❌ CRITICAL | 4 unfixed critical vulnerabilities |
| Functionality | ⚠️ AT RISK | Core logic has accounting errors |
| Testing | ✅ COVERAGE | POC tests for all findings |
| Documentation | ✅ COMPLETE | Comprehensive audit report |
| Ready for Mainnet | ❌ NO | Must fix all findings first |

### Audit Metadata

- **Audit Date:** August 2024
- **Branch:** security-audit
- **Repository:** https://github.com/anton-kovachev/2024-08-MyCut
- **Findings:** 4 Critical
- **Files Analyzed:** src/Pot.sol, src/ContestManager.sol, test/TestMyCut.t.sol
- **Documentation:** Complete with POC code and multiple fix options

---

### Next Steps

1. ✅ Review all audit findings in detail
2. 🔄 Implement recommended security fixes
3. 🧪 Run full test suite with POC tests
4. 📋 Get re-audit after implementing fixes
5. 🚀 Deploy only after security team approval

---

[//]: # (scope-open)

## Scope (contracts)

All Contracts in `src` are in scope.

```js
src/
├── ContestManager.sol
├── Pot.sol
```

## Compatibilities

- Blockchains: EVM Equivalent Chains Only
- Tokens: Standard ERC20 Tokens Only

**⚠️ AUDIT STATUS:** This code contains critical security vulnerabilities. See [AUDIT_README.md](./AUDIT_README.md) for complete security audit findings.

[//]: # (scope-close)

[//]: # (getting-started-open)

## Setup

Clone the repo
```bash
git clone https://github.com/Cyfrin/2024-08-MyCut.git
```
Open in VSCode
```bash
code 2024-08-MyCut/
```

Build and run tests
```bash
forge test
```

[//]: # (getting-started-close)

[//]: # (known-issues-open)

## Known Issues

⚠️ **CRITICAL - SECURITY AUDIT FINDINGS**

This code contains **4 CRITICAL vulnerabilities** identified in the security audit (August 2024):

1. **Missing Duplicate Player Validation** - Up to 100% reward loss
2. **Incorrect Bonus Distribution Calculation** - 60-90% bonus pool locked
3. **Missing Reward Sum Validation** - Contract insolvency risk
4. **Unchecked Token Transfer Return Value** - State corruption

**See [AUDIT_README.md](./AUDIT_README.md) and individual FINDING_*.md files for complete details.**

**DO NOT DEPLOY TO MAINNET without fixing all vulnerabilities.**

[//]: # (known-issues-close)
