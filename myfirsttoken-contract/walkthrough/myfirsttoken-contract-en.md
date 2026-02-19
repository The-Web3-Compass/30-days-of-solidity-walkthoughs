# MyFirstToken Contract: Creating Your Own Cryptocurrency

Welcome to one of the most powerful concepts in blockchain: **creating your own token**. Today, you're building an ERC-20 token - the same standard used by USDC ($30B market cap), LINK ($7B), UNI ($4B), and thousands of other tokens.

## Why Tokens Changed Everything

**Before tokens:** Only Bitcoin and Ethereum existed. If you wanted to create a new cryptocurrency, you had to:
- Fork an entire blockchain
- Set up mining infrastructure
- Build wallets from scratch
- Convince exchanges to list you
- Cost: Millions of dollars

**After ERC-20:** Anyone can create a token in minutes:
- Deploy one smart contract
- Works with all existing wallets (MetaMask, Ledger, etc.)
- Compatible with all DEXs (Uniswap, SushiSwap)
- Can be listed on exchanges immediately
- Cost: ~$50 in gas fees

**This democratized finance.** Suddenly, anyone could create:
- Governance tokens (UNI, AAVE, COMP)
- Stablecoins (USDC, DAI, USDT)
- Utility tokens (LINK, GRT, BAT)
- Meme coins (DOGE, SHIB, PEPE)

## The Problem ERC-20 Solved

**Early 2017:** Every token had different functions:
- Token A: `sendTokens(address, uint)`
- Token B: `transferCoins(address, uint)`  
- Token C: `move(address, uint)`

**The chaos:**
- Wallets couldn't support all tokens
- Exchanges had to integrate each token separately
- DApps couldn't work with multiple tokens
- Developers wasted time on basic functionality

**November 2015:** Fabian Vogelsteller proposed ERC-20 - a standard interface that all tokens should follow.

**The result:** Explosive growth. From dozens of tokens to thousands, all compatible with each other.

#### What Kind of Rules Does ERC-20 Define?

* Naming and Display: name, symbol, decimals.
* Balances and Supply: totalSupply(), balanceOf(address).
* Transfers: transfer() function.
* Approvals and Delegated Spending: approve() and transferFrom().
* Event Emissions: Transfer and Approval events.

### Minimal ERC-20 Token: SimpleERC20.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SimpleERC20 {
    string public name = "SimpleToken";
    string public symbol = "SIM";
    uint8 public decimals = 18;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(uint256 _initialSupply) {
        totalSupply = _initialSupply * (10 ** uint256(decimals));
        balanceOf[msg.sender] = totalSupply;
        emit Transfer(address(0), msg.sender, totalSupply);
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balanceOf[msg.sender] >= _value, "Not enough balance");
        _transfer(msg.sender, _to, _value);
        return true;
    }

    function approve(address _spender, uint256 _value) public returns (bool) {
        allowance[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) public returns (bool) {
        require(balanceOf[_from] >= _value, "Not enough balance");
        require(allowance[_from][msg.sender] >= _value, "Allowance too low");
        allowance[_from][msg.sender] -= _value;
        _transfer(_from, _to, _value);
        return true;
    }

    function _transfer(address _from, address _to, uint256 _value) internal {
        require(_to != address(0), "Invalid address");
        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        emit Transfer(_from, _to, _value);
    }
}
```

---

## Understanding ERC-20 Functions

### The Core Functions

**1. totalSupply()** - How many tokens exist?
```solidity
function totalSupply() public view returns (uint256);
```

**2. balanceOf()** - How many tokens does an address have?
```solidity
function balanceOf(address account) public view returns (uint256);
```

**3. transfer()** - Send tokens to someone
```solidity
function transfer(address to, uint256 amount) public returns (bool);
```

**4. approve()** - Allow someone to spend your tokens
```solidity
function approve(address spender, uint256 amount) public returns (bool);
```

**5. allowance()** - Check how much someone is allowed to spend
```solidity
function allowance(address owner, address spender) public view returns (uint256);
```

**6. transferFrom()** - Spend someone else's tokens (with permission)
```solidity
function transferFrom(address from, address to, uint256 amount) public returns (bool);
```

### The Approve/TransferFrom Pattern

This is the most confusing part of ERC-20 for beginners. Let me explain:

**Scenario:** You want to use Uniswap to swap your tokens.

**Step 1: Approve**
```solidity
// You call this on your token contract
token.approve(uniswapAddress, 1000);
// "Uniswap, you can spend up to 1000 of my tokens"
```

**Step 2: TransferFrom**
```solidity
// Uniswap calls this
token.transferFrom(yourAddress, uniswapAddress, 500);
// Uniswap moves 500 tokens from you to itself
```

**Why this pattern?** Security! Without it, contracts couldn't safely interact with tokens. The approve/transferFrom pattern lets you control exactly how much a contract can spend.

---

## Token Economics Concepts

### Decimals

```solidity
uint8 public decimals = 18;
```

**Why 18?** To match ETH's precision. 

**What does it mean?**
- 1 token = 1,000,000,000,000,000,000 (1e18) smallest units
- Like dollars and cents: $1.00 = 100 cents
- But with 18 decimal places instead of 2

**Example:**
- You want to send 1.5 tokens
- Actual amount: 1.5 × 10^18 = 1,500,000,000,000,000,000

**Why so many decimals?** Precision for DeFi calculations and micro-transactions.

### Initial Supply

```solidity
constructor(uint256 _initialSupply) {
    totalSupply = _initialSupply * (10 ** uint256(decimals));
    balanceOf[msg.sender] = totalSupply;
}
```

**Token distribution strategies:**

**1. Fixed Supply (Bitcoin model)**
- Create all tokens at once
- No more can ever be created
- Example: 21 million total

**2. Mintable (Ethereum model)**
- Start with some tokens
- Can create more later
- Usually controlled by governance

**3. Burnable**
- Tokens can be destroyed
- Reduces supply over time
- Deflationary model

---

## Real-World Token Examples

### USDC (Stablecoin)
- **Supply:** Unlimited (minted when USD deposited)
- **Decimals:** 6 (not 18!)
- **Special:** Pausable, blacklistable
- **Use case:** Stable value, payments

### UNI (Governance)
- **Supply:** 1 billion (fixed)
- **Decimals:** 18
- **Special:** Voting power, delegation
- **Use case:** Protocol governance

### LINK (Utility)
- **Supply:** 1 billion (fixed)
- **Decimals:** 18
- **Special:** Used to pay for oracle services
- **Use case:** Chainlink network fees

---

## Security Considerations

### 1. Integer Overflow (Pre-0.8.0)

**The problem:**
```solidity
// Solidity 0.7.x
uint8 balance = 255;
balance = balance + 1; // Wraps to 0!
```

**The fix:** Solidity 0.8.0+ has automatic overflow protection.

### 2. Approval Race Condition

**The problem:**
```solidity
// Alice approves Bob for 100 tokens
approve(bob, 100);

// Bob quickly spends 100 tokens
// Alice changes approval to 50
approve(bob, 50);

// Bob front-runs and spends another 50
// Total spent: 150 (more than intended!)
```

**The fix:**
```solidity
// Set to 0 first, then set new amount
approve(bob, 0);
approve(bob, 50);
```

### 3. Transfer to Zero Address

**The problem:**
```solidity
transfer(address(0), 1000); // Tokens lost forever!
```

**The fix:**
```solidity
require(_to != address(0), "Invalid address");
```

---

## Advanced Token Features

### Minting

```solidity
function mint(address to, uint256 amount) public onlyOwner {
    totalSupply += amount;
    balanceOf[to] += amount;
    emit Transfer(address(0), to, amount);
}
```

### Burning

```solidity
function burn(uint256 amount) public {
    require(balanceOf[msg.sender] >= amount);
    balanceOf[msg.sender] -= amount;
    totalSupply -= amount;
    emit Transfer(msg.sender, address(0), amount);
}
```

### Pausable

```solidity
bool public paused;

modifier whenNotPaused() {
    require(!paused, "Token is paused");
    _;
}

function transfer(address to, uint256 amount) public whenNotPaused returns (bool) {
    // Transfer logic
}
```

---

## Using OpenZeppelin (Production Standard)

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, Ownable {
    constructor(uint256 initialSupply) ERC20("MyToken", "MTK") Ownable(msg.sender) {
        _mint(msg.sender, initialSupply * 10 ** decimals());
    }
    
    // Add minting capability
    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}
```

**What OpenZeppelin gives you:**
- Battle-tested code
- Gas-optimized
- Security audited
- Industry standard
- Extensions (burnable, pausable, snapshot, etc.)

---

## Key Concepts You've Learned

**1. ERC-20 standard** - The interface that makes tokens compatible

**2. Approve/TransferFrom** - How contracts interact with tokens

**3. Decimals** - Precision in token amounts

**4. Token economics** - Supply models and distribution

**5. Security patterns** - Preventing common vulnerabilities

**6. OpenZeppelin** - Production-ready implementations

---

## Why This Matters

You just learned how to create:
- **Governance tokens** - For DAOs
- **Utility tokens** - For protocols
- **Stablecoins** - For payments
- **Reward tokens** - For incentives

**Real impact:** The ERC-20 standard enabled:
- ICO boom (2017)
- DeFi summer (2020)
- NFT explosion (2021)
- Current Web3 ecosystem

## Challenge Yourself

Extend your token:
1. Add minting and burning capabilities
2. Implement a vesting schedule for team tokens
3. Create a token with transaction fees
4. Build a deflationary token (burns on transfer)
5. Add snapshot functionality for voting

You've mastered token creation. This is the foundation of the entire crypto economy!
