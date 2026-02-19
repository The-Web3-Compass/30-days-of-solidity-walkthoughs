# Automated Market Maker: Building the Heart of DeFi

Welcome to one of the most important contracts in all of Web3! Today you're building an **Automated Market Maker (AMM)** - the technology that powers Uniswap, SushiSwap, PancakeSwap, and countless other decentralized exchanges.

## The Problem with Traditional Exchanges

Traditional exchanges (Binance, Coinbase, even stock markets) use an **order book**:
- Alice wants to sell 10 ETH at $2,000 each
- Bob wants to buy 5 ETH at $1,950 each
- The exchange matches buyers and sellers

This works, but it has problems:
- **Requires liquidity providers** - Someone has to actively place orders
- **Centralized** - The exchange controls the order book
- **Slow** - Matching orders takes time
- **Expensive** - Maintaining order books on-chain would cost a fortune in gas

## The AMM Solution

AMMs flip the script. Instead of matching individual buyers and sellers, we create a **liquidity pool** - a big pot of two tokens that anyone can trade against using a simple mathematical formula.

**The magic formula: x × y = k**

- `x` = Amount of Token A in the pool
- `y` = Amount of Token B in the pool  
- `k` = A constant that never changes during swaps

**Here's the genius:** If you want to buy Token A (removing it from the pool), you must add Token B to keep `k` constant. The more you buy, the more expensive it gets. This automatically creates a price curve!

Let me show you with numbers:

**Starting state:**
- Pool has 100 Token A and 100 Token B
- k = 100 × 100 = 10,000

**You want to buy 10 Token A:**
- New x = 90 (you removed 10)
- To keep k = 10,000, we need: 90 × y = 10,000
- So y = 111.11
- You must add 11.11 Token B

**The price changed!** Initially, the ratio was 1:1. But after your trade, it's 90:111.11 (Token A became more expensive). This is called **slippage** and it's how AMMs naturally adjust prices based on supply and demand.

### Full Contract: AutomatedMarketMaker.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract AutomatedMarketMaker is ERC20 {
    IERC20 public tokenA;
    IERC20 public tokenB;
    uint256 public reserveA;
    uint256 public reserveB;

    constructor(address _tokenA, address _tokenB, string memory name, string memory symbol) ERC20(name, symbol) {
        tokenA = IERC20(_tokenA);
        tokenB = IERC20(_tokenB);
    }

    function addLiquidity(uint256 amountA, uint256 amountB) external {
        tokenA.transferFrom(msg.sender, address(this), amountA);
        tokenB.transferFrom(msg.sender, address(this), amountB);
        
        uint256 liquidity;
        if (totalSupply() == 0) {
            liquidity = sqrt(amountA * amountB);
        } else {
            liquidity = min(
                (amountA * totalSupply()) / reserveA,
                (amountB * totalSupply()) / reserveB
            );
        }
        
        _mint(msg.sender, liquidity);
        reserveA += amountA;
        reserveB += amountB;
    }

    function swapAforB(uint256 amountAIn) external {
        // 1. Calculate price
        // Input with 0.3% fee
        uint256 amountAInWithFee = (amountAIn * 997) / 1000;
        
        // Calculate amount out (y = k / x)
        // (reserveA + amountIn) * (reserveB - amountOut) = k
        uint256 amountBOut = (reserveB * amountAInWithFee) / (reserveA + amountAInWithFee);

        // 2. Transfer Tokens
        tokenA.transferFrom(msg.sender, address(this), amountAIn);
        tokenB.transfer(msg.sender, amountBOut);

        // 3. Update Reserves
        reserveA += amountAIn; // Note: Reserves track raw balance, so we add full amount (fee included in pool)
        reserveB -= amountBOut;
    }

    // Math Helpers
    function sqrt(uint256 y) internal pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) { z = x; x = (y / x + x) / 2; }
        } else if (y != 0) { z = 1; }
    }

    function min(uint256 a, uint256 b) internal pure returns (uint256) { return a < b ? a : b; }
}
```

---

## Breaking Down the Contract

### Why This Contract is Also an ERC20 Token

```solidity
contract AutomatedMarketMaker is ERC20 {
```

Wait, our AMM **inherits from ERC20**? That's right! When you provide liquidity to the pool, you get **LP tokens** (Liquidity Provider tokens) in return. These tokens:
- Represent your share of the pool
- Can be traded or transferred
- Can be redeemed later to get your liquidity back (plus fees earned!)

This is how Uniswap works. When you add liquidity to a pool, you get UNI-V2 tokens representing your ownership stake.

---

## The Constructor: Setting Up the Pool

```solidity
constructor(address _tokenA, address _tokenB, string memory name, string memory symbol) ERC20(name, symbol) {
    tokenA = IERC20(_tokenA);
    tokenB = IERC20(_tokenB);
}
```

**What's happening:**
- We're creating a pool for two specific tokens (e.g., ETH/USDC)
- `IERC20` is an interface - it lets us interact with any ERC20 token
- We also initialize our LP token with a name and symbol (e.g., "ETH-USDC LP")

---

## Adding Liquidity: Becoming a Liquidity Provider

This is how the pool gets funded:

```solidity
function addLiquidity(uint256 amountA, uint256 amountB) external {
    // 1. Transfer tokens from user to this contract
    tokenA.transferFrom(msg.sender, address(this), amountA);
    tokenB.transferFrom(msg.sender, address(this), amountB);
    
    uint256 liquidity;
    if (totalSupply() == 0) {
        // First liquidity provider
        liquidity = sqrt(amountA * amountB);
    } else {
        // Subsequent providers get proportional share
        liquidity = min(
            (amountA * totalSupply()) / reserveA,
            (amountB * totalSupply()) / reserveB
        );
    }
    
    // 2. Mint LP tokens to the provider
    _mint(msg.sender, liquidity);
    
    // 3. Update reserves
    reserveA += amountA;
    reserveB += amountB;
}
```

**Step-by-step breakdown:**

### Step 1: Transfer Tokens to the Pool

```solidity
tokenA.transferFrom(msg.sender, address(this), amountA);
tokenB.transferFrom(msg.sender, address(this), amountB);
```

You're depositing both tokens into the pool. The pool now holds them and will use them for swaps.

**Important:** Before calling this, you must `approve()` the AMM contract to spend your tokens!

### Step 2: Calculate LP Tokens to Mint

**For the first depositor:**
```solidity
liquidity = sqrt(amountA * amountB);
```

Why square root? It's a geometric mean that prevents manipulation. If you deposit 100 Token A and 100 Token B, you get `sqrt(100 * 100) = 100` LP tokens.

**For subsequent depositors:**
```solidity
liquidity = min(
    (amountA * totalSupply()) / reserveA,
    (amountB * totalSupply()) / reserveB
);
```

This ensures you get a fair share proportional to what you're adding. We use `min()` to prevent someone from manipulating the pool by adding unbalanced liquidity.

**Example:**
- Pool has 1000 Token A, 1000 Token B, and 1000 LP tokens issued
- You add 100 Token A and 100 Token B
- You get: `min((100 * 1000) / 1000, (100 * 1000) / 1000) = 100` LP tokens
- You now own 100/1100 = 9.09% of the pool

### Step 3: Mint LP Tokens and Update Reserves

```solidity
_mint(msg.sender, liquidity);
reserveA += amountA;
reserveB += amountB;
```

You receive your LP tokens, and the pool's reserves are updated. You're now earning fees from every swap!

---

## The Swap Function: Where the Magic Happens

This is the core of the AMM:

```solidity
function swapAforB(uint256 amountAIn) external {
    // 1. Calculate output amount with 0.3% fee
    uint256 amountAInWithFee = (amountAIn * 997) / 1000;
    
    // 2. Apply the constant product formula
    uint256 amountBOut = (reserveB * amountAInWithFee) / (reserveA + amountAInWithFee);

    // 3. Execute the swap
    tokenA.transferFrom(msg.sender, address(this), amountAIn);
    tokenB.transfer(msg.sender, amountBOut);

    // 4. Update reserves
    reserveA += amountAIn;
    reserveB -= amountBOut;
}
```

**Let's break down the math:**

### The 0.3% Fee

```solidity
uint256 amountAInWithFee = (amountAIn * 997) / 1000;
```

If you're swapping 1000 Token A:
- Fee: 1000 × 0.003 = 3 tokens
- Amount after fee: 1000 × 0.997 = 997 tokens

This fee stays in the pool and is distributed to liquidity providers. That's how they earn money!

### The Constant Product Formula

```solidity
uint256 amountBOut = (reserveB * amountAInWithFee) / (reserveA + amountAInWithFee);
```

This enforces `x × y = k`. Let's derive it:

**Before swap:**
- `k = reserveA × reserveB`

**After swap:**
- New reserveA = `reserveA + amountAIn`
- New reserveB = `reserveB - amountBOut`
- Must maintain: `(reserveA + amountAIn) × (reserveB - amountBOut) = k`

**Solving for amountBOut:**
```
(reserveA + amountAIn) × (reserveB - amountBOut) = reserveA × reserveB
reserveB - amountBOut = (reserveA × reserveB) / (reserveA + amountAIn)
amountBOut = reserveB - (reserveA × reserveB) / (reserveA + amountAIn)
amountBOut = (reserveB × amountAIn) / (reserveA + amountAIn)
```

**That's the formula!** It automatically calculates the fair price based on the current reserves.

### Price Impact Example

**Starting state:**
- 1000 Token A, 1000 Token B in pool
- k = 1,000,000

**You swap 100 Token A (after 0.3% fee = 99.7):**
- amountBOut = (1000 × 99.7) / (1000 + 99.7) = 90.66 Token B
- New state: 1099.7 Token A, 909.34 Token B

**The price changed!**
- Before: 1 Token A = 1 Token B
- After: 1 Token A ≈ 0.827 Token B (Token A got cheaper because there's more of it)

This is **slippage** - the larger your trade relative to the pool, the worse your price. This incentivizes people to split large trades or add more liquidity!

---

## The Math Helper Functions

### Square Root Function

```solidity
function sqrt(uint256 y) internal pure returns (uint256 z) {
    if (y > 3) {
        z = y;
        uint256 x = y / 2 + 1;
        while (x < z) { z = x; x = (y / x + x) / 2; }
    } else if (y != 0) { z = 1; }
}
```

This uses the **Babylonian method** (also called Newton's method) to calculate square roots. Solidity doesn't have a built-in sqrt function, so we implement it ourselves.

**How it works:** It makes better and better guesses until it converges on the answer. For `sqrt(100)`, it might go: 50 → 26 → 14 → 10.

### Min Function

```solidity
function min(uint256 a, uint256 b) internal pure returns (uint256) { 
    return a < b ? a : b; 
}
```

Simple ternary operator - returns the smaller of two numbers.

---

## Why This Changed Finance Forever

You just implemented the algorithm that powers billions of dollars in daily trading volume. Here's why it's revolutionary:

**1. Permissionless** - Anyone can create a pool for any token pair
**2. Always available** - No need to wait for someone to match your order
**3. Transparent** - All prices are calculated on-chain, no hidden manipulation
**4. Passive income** - Liquidity providers earn fees automatically
**5. Composable** - Other contracts can integrate with AMMs programmatically

---

## Real-World Considerations

**What's missing from this simplified version:**

1. **Slippage protection** - Users should specify minimum output amount
2. **Deadline** - Transactions should expire if not mined quickly
3. **Remove liquidity** - LP token holders need a way to withdraw
4. **Price oracles** - Track time-weighted average prices for other contracts
5. **Flash loan protection** - Prevent manipulation within a single transaction

---

## Key Concepts You've Mastered

- **Constant product formula** (x × y = k)
- **Liquidity provision** and LP tokens
- **Automated pricing** based on supply and demand
- **Fee mechanisms** for incentivizing liquidity providers
- **Slippage** and price impact

This is the foundation of DeFi. Uniswap V2 is essentially this contract with more features. You now understand how decentralized exchanges work at a fundamental level.

## Challenge Yourself

Extend this AMM:
1. Add a `removeLiquidity()` function that burns LP tokens
2. Implement slippage protection (minimum output amount)
3. Add a `swapBforA()` function for the reverse direction
4. Create a price oracle using time-weighted averages
5. Add events for all major actions

You've just built the engine that powers modern DeFi. This is huge!
