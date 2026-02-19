# Yield Farming Platform: Building DeFi Incentive Systems

Welcome to the heart of DeFi: **yield farming**. Today, you're building the mechanism that powers billions in liquidity incentives - the same system used by Uniswap, Aave, Compound, and every major protocol.

## The DeFi Liquidity Problem

**The challenge:**
- You launch a new DeFi protocol
- Need liquidity to function
- Users won't deposit without incentive
- Chicken and egg problem

**The solution:** Reward users for providing liquidity.

**How it works:**
1. Users stake tokens (provide liquidity)
2. Earn rewards over time (yield)
3. Claim rewards whenever they want
4. Unstake to get original tokens back

**Real-world example:**
- Uniswap V2: Stake LP tokens, earn UNI
- Compound: Supply assets, earn COMP
- Aave: Deposit collateral, earn AAVE

**The result:** $100B+ in Total Value Locked (TVL) across DeFi.

### Big Picture Logic
1. Staking: Users deposit tokens.
2. Earning: Rewards calculated second-by-second.
3. Claiming: Harvest accumulated rewards.
4. Emergency Withdraw: Instant exit, lose pending rewards.

### Full Yield Farming Contract
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract YieldFarming is ReentrancyGuard {
    IERC20 public stakingToken;
    IERC20 public rewardToken;
    uint256 public rewardRatePerSecond;
    // ... (full implementation including stake, unstake, claimRewards functions)
}
```

---

## Understanding Reward Calculations

### Time-Based Rewards

```solidity
uint256 public rewardRatePerSecond = 1e18; // 1 token per second

function calculateReward(address user) public view returns (uint256) {
    uint256 timeStaked = block.timestamp - stakeTimestamp[user];
    uint256 stakedAmount = stakedBalance[user];
    return (stakedAmount * timeStaked * rewardRatePerSecond) / 1e18;
}
```

**Example:**
- User stakes 100 tokens
- Rate: 0.01 tokens per second per token staked
- After 1 day (86,400 seconds):
- Reward = 100 * 86,400 * 0.01 = 86,400 tokens

### APY Calculation

**Annual Percentage Yield (APY):**
```solidity
function calculateAPY() public view returns (uint256) {
    uint256 yearlyReward = rewardRatePerSecond * 365 days;
    uint256 totalStaked = stakingToken.balanceOf(address(this));
    return (yearlyReward * 100) / totalStaked; // Percentage
}
```

**Example:**
- 1,000,000 tokens staked
- 100,000 tokens rewarded per year
- APY = (100,000 / 1,000,000) * 100 = 10%

---

## Key Concepts You've Learned

**1. Staking mechanics** - Locking tokens for rewards

**2. Time-based rewards** - Calculating earnings per second

**3. Reward distribution** - Fair allocation to stakers

**4. Emergency withdrawal** - Exit without rewards

**5. APY calculation** - Understanding yield rates

**6. Reentrancy protection** - Secure token transfers

---

## Advanced Yield Farming Features

### 1. Boosted Rewards (Lock Duration)

```solidity
function calculateBoostedReward(address user) public view returns (uint256) {
    uint256 baseReward = calculateReward(user);
    uint256 lockDuration = lockEndTime[user] - stakeTimestamp[user];

    if (lockDuration >= 365 days) {
        return baseReward * 2; // 2x for 1 year lock
    } else if (lockDuration >= 180 days) {
        return baseReward * 15 / 10; // 1.5x for 6 months
    }
    return baseReward;
}
```

### 2. Multiple Reward Tokens

```solidity
IERC20[] public rewardTokens;
mapping(address => uint256) public rewardRates;

function claimAllRewards() external {
    for (uint i = 0; i < rewardTokens.length; i++) {
        uint256 reward = calculateReward(msg.sender, address(rewardTokens[i]));
        rewardTokens[i].transfer(msg.sender, reward);
    }
}
```

### 3. Penalty for Early Withdrawal

```solidity
function withdrawWithPenalty() external {
    uint256 stakedAmount = stakedBalance[msg.sender];
    uint256 penalty = stakedAmount * 10 / 100; // 10% penalty

    stakedBalance[msg.sender] = 0;
    stakingToken.transfer(msg.sender, stakedAmount - penalty);
    stakingToken.transfer(treasury, penalty);
}
```

---

## Real-World Yield Farming Protocols

### Uniswap V2 Liquidity Mining

**How it works:**
- Provide liquidity to pools (get LP tokens)
- Stake LP tokens in farming contract
- Earn UNI tokens as rewards
- Claim anytime, unstake anytime

**Result:** $10B+ in liquidity attracted

### Compound COMP Distribution

**How it works:**
- Supply or borrow assets
- Earn COMP tokens automatically
- Distributed per block based on usage
- No staking required

**Innovation:** Rewards for both suppliers AND borrowers

### Curve Finance veCRV

**How it works:**
- Lock CRV tokens for up to 4 years
- Get veCRV (vote-escrowed CRV)
- Earn boosted rewards (up to 2.5x)
- Voting power in governance

**Innovation:** Time-locked tokens for better alignment

---

## Security Considerations

### 1. Reward Manipulation

**Problem:** Users stake right before reward distribution, unstake after.

**Solution:** Minimum staking period.
```solidity
require(block.timestamp >= stakeTimestamp[msg.sender] + 7 days, "Too soon");
```

### 2. Reward Depletion

**Problem:** Rewards run out, users can't claim.

**Solution:** Monitor reward pool balance.
```solidity
function checkRewardSolvency() public view returns (bool) {
    uint256 totalPendingRewards = calculateTotalPendingRewards();
    return rewardToken.balanceOf(address(this)) >= totalPendingRewards;
}
```

### 3. Flash Loan Attacks

**Problem:** Borrow huge amount, stake, claim rewards, repay.

**Solution:** Time-weighted rewards or minimum lock period.

---

## Why This Matters

Yield farming enabled:
- **DeFi Summer 2020** - $10B+ TVL growth
- **Protocol bootstrapping** - Cold start problem solved
- **User acquisition** - Incentivized participation
- **Liquidity provision** - Deep markets
- **Token distribution** - Fair launch mechanism

**Current DeFi TVL:** $50B+ across protocols, mostly from yield farming.

## Challenge Yourself

Extend this yield farming platform:
1. Add boosted rewards for longer lock periods
2. Implement multiple reward tokens
3. Create tiered rewards based on stake size
4. Add referral bonuses for bringing new users
5. Build a governance system with staked tokens

You've mastered yield farming mechanics. This is the foundation of modern DeFi!

---

**Congratulations! You've completed all 30 Days of Solidity!** 🎉

You've learned:
- Smart contract fundamentals
- DeFi protocols (lending, DEXs, stablecoins)
- NFT standards and marketplaces
- Security patterns and best practices
- Gas optimization techniques
- Upgradeable contracts
- Real-world production patterns

**You're now ready to build production Web3 applications!**
