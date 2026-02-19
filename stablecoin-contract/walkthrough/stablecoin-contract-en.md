# SimpleStablecoin: Building Decentralized Stable Currency

Welcome to one of DeFi's most important innovations: **algorithmic stablecoins**. Today, you're building a collateralized stablecoin system - the same mechanism that powers MakerDAO's DAI ($5B+ market cap).

## The Volatility Problem

**Crypto's biggest issue:**
- Bitcoin: $69K → $16K (77% drop)
- Ethereum: $4.8K → $900 (81% drop)
- Impossible to use for payments
- Can't hold value reliably

**What we need:** A cryptocurrency that stays at $1.

## Types of Stablecoins

### 1. Fiat-Backed (USDC, USDT)
- **Mechanism:** 1 token = 1 USD in bank
- **Pros:** Simple, stable
- **Cons:** Centralized, requires trust, can be frozen

### 2. Crypto-Backed (DAI, our contract)
- **Mechanism:** Over-collateralized with crypto
- **Pros:** Decentralized, trustless
- **Cons:** Capital inefficient, complex

### 3. Algorithmic (failed: UST)
- **Mechanism:** Algorithm adjusts supply
- **Pros:** Capital efficient
- **Cons:** Death spiral risk (UST lost $40B)

**We're building #2** - the proven, safe approach.

## How Over-Collateralization Works

**The core principle:** Lock $150 of ETH to mint $100 of stablecoin.

**Why 150%?** Safety buffer for price volatility.

**Example:**
1. ETH = $2,000
2. You deposit 1 ETH ($2,000)
3. You can mint up to $1,333 of stablecoin (150% ratio)
4. ETH drops to $1,500
5. Your collateral is still $1,500 (>$1,333 debt)
6. System remains solvent

**If ETH drops below 150%?** Liquidation - your collateral is sold to cover the debt.

### SimpleStablecoin.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract SimpleStablecoin is ERC20, ReentrancyGuard, Ownable {
    // 1. The Oracle (To know the price of ETH)
    AggregatorV3Interface internal priceFeed;

    // 2. The Ratio (150% Collateral needed)
    uint256 public constant COLLATERAL_RATIO = 150; 

    // User -> Amount of ETH deposited
    mapping(address => uint256) public collateralDeposited;

    constructor(address _priceFeedAddress) ERC20("StableUSD", "SUSD") {
        priceFeed = AggregatorV3Interface(_priceFeedAddress);
    }

    // --- MAIN FUNCTIONS ---

    function depositCollateral() external payable {
        collateralDeposited[msg.sender] += msg.value;
    }

    function mintStablecoin(uint256 amountToMint) external nonReentrant {
        uint256 currentEthValue = getCollateralValueInUsd(msg.sender);
        uint256 currentDebt = balanceOf(msg.sender);
        
        // Check health factor
        uint256 maxMintable = (currentEthValue * 100) / COLLATERAL_RATIO;
        require(currentDebt + amountToMint <= maxMintable, "Not enough collateral!");

        _mint(msg.sender, amountToMint);
    }

    function burnStablecoin(uint256 amountToBurn) external nonReentrant {
        _burn(msg.sender, amountToBurn);
    }

    function withdrawCollateral(uint256 amount) external nonReentrant {
        uint256 currentDebt = balanceOf(msg.sender);
        
        // Calculate remaining collateral value AFTER withdrawal
        uint256 remainingCollateral = collateralDeposited[msg.sender] - amount;
        uint256 remainingValue = (remainingCollateral * getEthPrice()) / 1e18; // Price is 8 decimals usually, but let's assume standard formatting for tutorial simplicity

        uint256 requiredCollateralValue = (currentDebt * COLLATERAL_RATIO) / 100;

        require(remainingValue >= requiredCollateralValue, "Cannot withdraw, health factor too low");

        collateralDeposited[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }

    // --- ORACLE MAGIC ---
    
    function getEthPrice() public view returns (uint256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();
        // Chainlink returns price with 8 decimals (e.g. 200000000000 for $2000)
        // We want everything in 18 decimals like ETH
        return uint256(price) * 1e10; 
    }

    function getCollateralValueInUsd(address user) public view returns (uint256) {
        uint256 ethAmount = collateralDeposited[user];
        uint256 ethPrice = getEthPrice();
        // (ETH Amount * Price) / 1e18
        return (ethAmount * ethPrice) / 1e18;
    }
}
```

---

### The Economic Model

#### 1. The Oracle
We can't just trust the user to say "ETH is worth $5000". We ask Chainlink.
```solidity
priceFeed.latestRoundData();
```

#### 2. The Golden Rule: 150% Ratio
If you want **$100** of StableUSD, you must lock **$150** of ETH.
Why? Because if ETH drops by 10%, you still have $135 of collateral backing the $100 debt. The system stays solvent.

#### 3. Minting
When you mint, you are creating money out of thin air... backed by your ETH vault.
```solidity
_mint(msg.sender, amount);
```

#### 4. Withdrawal
You can take your ETH back anytime, **as long as** you pay back the debt (burn the tokens) or if the ETH price has gone up enough that you have "extra" collateral.

This is the exact mechanism that powers **MakerDAO (DAI)** and many other DeFi giants.

---

## Understanding Liquidations

### Why Liquidations Are Necessary

**The problem:** ETH drops 40%, collateral now worth less than debt.

**Without liquidations:** System becomes insolvent, stablecoin loses peg.

**With liquidations:** Undercollateralized positions are closed, system stays healthy.

### Liquidation Mechanics

```solidity
function liquidate(address user) external {
    uint256 collateralValue = getCollateralValueInUsd(user);
    uint256 debtValue = balanceOf(user);
    uint256 healthFactor = (collateralValue * 100) / debtValue;
    
    require(healthFactor < COLLATERAL_RATIO, "Position is healthy");
    
    // Liquidator pays debt, gets collateral + bonus
    uint256 liquidationBonus = collateralDeposited[user] * 5 / 100; // 5% bonus
    
    _burn(msg.sender, debtValue);
    collateralDeposited[user] = 0;
    payable(msg.sender).transfer(collateralDeposited[user] + liquidationBonus);
}
```

**Liquidation incentive:** Liquidators earn 5-10% bonus for keeping system healthy.

---

## Key Concepts You've Learned

**1. Stablecoins** - Cryptocurrencies pegged to fiat

**2. Over-collateralization** - Safety through excess collateral

**3. Collateral ratio** - 150% minimum for safety

**4. Oracle integration** - Chainlink for price feeds

**5. Minting/burning** - Creating and destroying tokens

**6. Liquidations** - Protecting system solvency

---

## Real-World Stablecoin Protocols

### MakerDAO (DAI)

**Features:**
- $5B+ market cap
- Multiple collateral types
- Decentralized governance
- Stability fee (interest)
- Emergency shutdown mechanism

**Collateral ratio:** 150-175% depending on asset

### Liquity (LUSD)

**Features:**
- ETH-only collateral
- 110% minimum ratio
- No governance
- One-time fee

**Innovation:** More capital efficient (lower ratio)

### Frax

**Features:**
- Partially algorithmic
- Partially collateralized
- Hybrid approach
- AMO (Algorithmic Market Operations)

---

## The UST Collapse (Cautionary Tale)

**What happened (May 2022):**
1. UST was algorithmic (not collateralized)
2. Relied on LUNA token for stability
3. Large withdrawals triggered death spiral
4. UST lost peg ($1 → $0.10)
5. $40B+ in value destroyed

**Lesson:** Over-collateralization is safer than algorithmic approaches.

---

## Why This Matters

Stablecoins enable:
- **DeFi** - Stable trading pairs, lending
- **Payments** - Crypto without volatility
- **Savings** - Earn yield on stable assets
- **Remittances** - Cross-border transfers
- **Store of value** - Hedge against inflation

**Market size:** $150B+ in stablecoins (mostly USDT, USDC, DAI)

## Challenge Yourself

Extend this stablecoin:
1. Add liquidation functionality with incentives
2. Support multiple collateral types (WBTC, LINK)
3. Implement a stability fee (interest on debt)
4. Add emergency shutdown mechanism
5. Create a governance token for parameter changes

You've mastered stablecoin mechanics. This is cutting-edge DeFi development!

---

**You just built the foundation of decentralized stable currency!**
