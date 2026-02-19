# PreorderTokens Contract: Building Token Sales and ICOs

Welcome to token distribution mechanics! You've built an ERC-20 token, but now you need to sell it to users. Today, you're building an **Initial Coin Offering (ICO)** or presale contract - the technology that powered the 2017 crypto boom and raised billions of dollars.

## The Token Distribution Problem

**You've created a token. Now what?**

**Options:**
1. **Airdrop** - Give tokens away for free (expensive, attracts farmers)
2. **Manual distribution** - Send tokens one by one (doesn't scale)
3. **Token sale** - Users buy tokens with ETH (automated, scalable)

**The challenge:** How do you prevent early buyers from immediately dumping and crashing the price?

## The ICO Boom (2017)

**Famous ICOs:**
- **Ethereum** - Raised $18M in 2014 (now worth billions)
- **EOS** - Raised $4.1B in 2018 (largest ICO ever)
- **Filecoin** - Raised $257M in 2017
- **Tezos** - Raised $232M in 2017

**The problem:** Many ICOs had no vesting or lock-up periods. Result:
1. Token sale completes
2. Early buyers immediately sell
3. Price crashes 90%+
4. Project dies

**The solution:** Lock tokens until sale ends, preventing immediate dumps.

### SimplifiedTokenSale.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./SimpleERC20.sol";

contract SimplifiedTokenSale is SimpleERC20 {
    uint256 public tokenPrice;
    uint256 public saleStartTime;
    uint256 public saleEndTime;
    address public projectOwner;
    bool public finalized = false;

    event TokensPurchased(address indexed buyer, uint256 etherAmount, uint256 tokenAmount);

    constructor(
        uint256 _initialSupply,
        uint256 _tokenPrice,
        uint256 _saleDurationInSeconds,
        address _projectOwner
    ) SimpleERC20(_initialSupply) {
        tokenPrice = _tokenPrice;
        saleStartTime = block.timestamp;
        saleEndTime = block.timestamp + _saleDurationInSeconds;
        projectOwner = _projectOwner;
        
        // Move all tokens to this contract so it can sell them
        _transfer(msg.sender, address(this), totalSupply);
    }

    // 1. BUYING MECHANISM
    function buyTokens() public payable {
        require(!finalized && block.timestamp <= saleEndTime, "Sale inactive");
        
        // Calculate amount: (ETH sent * decimals) / price
        uint256 tokenAmount = (msg.value * 10**uint256(decimals)) / tokenPrice;
        
        _transfer(address(this), msg.sender, tokenAmount);
        emit TokensPurchased(msg.sender, msg.value, tokenAmount);
    }

    // 2. LOCKING MECHANISM (Inheritance Magic!)
    // We override the transfer function of the parent ERC20.
    function transfer(address _to, uint256 _value) public override returns (bool) {
        // Only allow transfers if the sale is finalized OR if the contract itself is sending (for buying)
        require(finalized || msg.sender == address(this), "Tokens locked");
        return super.transfer(_to, _value);
    }

    // 3. WITHDRAWAL
    function finalizeSale() public {
        require(msg.sender == projectOwner && block.timestamp > saleEndTime, "Cannot finalize");
        finalized = true; // Unlocks transfers!
        payable(projectOwner).transfer(address(this).balance);
    }

    // Allow receiving ETH directly
    receive() external payable { buyTokens(); }
}
```

---

### Key Concepts

#### Inheritance and Overriding
We are inheriting from `SimpleERC20`. This gives us all the standard functions (`transfer`, `balanceOf`).
But we want to change how `transfer` works.
```solidity
contract SimplifiedTokenSale is SimpleERC20 { ... }
```
By adding `override` to the transfer function, we intercept every attempt to move tokens.
```solidity
function transfer(...) public override returns (bool) {
    require(finalized == true, "Tokens locked due to ongoing sale");
    return super.transfer(...); // Call the original logic
}
```

#### Time-Based Logic

In Solidity, `block.timestamp` gives us the current time (in seconds since 1970). We use this to set deadlines.

```solidity
saleEndTime = block.timestamp + _saleDurationInSeconds;

require(block.timestamp <= saleEndTime, "Sale is over!");
```

**Time units in Solidity:**
- `1 seconds` = 1
- `1 minutes` = 60
- `1 hours` = 3600
- `1 days` = 86400
- `1 weeks` = 604800

**Example:**
```solidity
// 7-day sale
saleEndTime = block.timestamp + 7 days;

// 30-day sale
saleEndTime = block.timestamp + 30 days;
```

#### The "Buy" Logic

It's simple math, but with a twist for decimals.

**Example calculation:**
- Token price: 0.01 ETH per token
- User sends: 1 ETH
- Tokens received: 1 / 0.01 = 100 tokens

**In Solidity:**
```solidity
uint256 tokenAmount = (msg.value * 10**decimals) / tokenPrice;
```

**Why multiply by `10**decimals`?**

Solidity doesn't have decimals/floats. Everything is an integer.

**Example with 18 decimals:**
- User sends: 1 ETH = 1,000,000,000,000,000,000 wei
- Token price: 0.01 ETH = 10,000,000,000,000,000 wei
- Calculation: (1e18 * 1e18) / 1e16 = 100e18 (100 tokens)

---

## Understanding Token Locking

### The Override Pattern

```solidity
function transfer(address _to, uint256 _value) public override returns (bool) {
    require(finalized || msg.sender == address(this), "Tokens locked");
    return super.transfer(_to, _value);
}
```

**What's happening:**

1. **Override** - We replace the parent's `transfer` function
2. **Check** - Only allow transfers if sale is finalized OR contract is sending
3. **Super** - Call the original transfer logic

**Why allow contract transfers?** So the contract can send tokens to buyers during the sale.

**Flow:**
```
User buys → Contract transfers (allowed) → User receives tokens
User tries to transfer → Blocked until finalized
Sale ends → Owner finalizes → Users can now transfer
```

---

## Advanced Token Sale Features

### 1. Tiered Pricing (Early Bird Discount)

```solidity
function getTokenPrice() public view returns (uint256) {
    uint256 elapsed = block.timestamp - saleStartTime;
    
    if (elapsed < 1 days) {
        return tokenPrice * 80 / 100; // 20% discount first day
    } else if (elapsed < 7 days) {
        return tokenPrice * 90 / 100; // 10% discount first week
    } else {
        return tokenPrice; // Full price
    }
}

function buyTokens() public payable {
    uint256 currentPrice = getTokenPrice();
    uint256 tokenAmount = (msg.value * 10**decimals) / currentPrice;
    _transfer(address(this), msg.sender, tokenAmount);
}
```

### 2. Hard Cap (Maximum Raise)

```solidity
uint256 public constant HARD_CAP = 1000 ether;
uint256 public totalRaised;

function buyTokens() public payable {
    require(totalRaised + msg.value <= HARD_CAP, "Hard cap reached");
    totalRaised += msg.value;
    // ... rest of logic
}
```

### 3. Whitelist (Private Sale)

```solidity
mapping(address => bool) public whitelist;

function addToWhitelist(address[] memory addresses) public {
    require(msg.sender == projectOwner);
    for (uint i = 0; i < addresses.length; i++) {
        whitelist[addresses[i]] = true;
    }
}

function buyTokens() public payable {
    require(whitelist[msg.sender], "Not whitelisted");
    // ... rest of logic
}
```

### 4. Vesting Schedule

```solidity
struct VestingSchedule {
    uint256 totalAmount;
    uint256 released;
    uint256 startTime;
    uint256 duration;
}

mapping(address => VestingSchedule) public vesting;

function releaseVested() public {
    VestingSchedule storage schedule = vesting[msg.sender];
    uint256 elapsed = block.timestamp - schedule.startTime;
    uint256 vested = (schedule.totalAmount * elapsed) / schedule.duration;
    uint256 releasable = vested - schedule.released;
    
    require(releasable > 0, "Nothing to release");
    schedule.released += releasable;
    _transfer(address(this), msg.sender, releasable);
}
```

---

## Security Considerations

### 1. Reentrancy in Refunds

**If you add refund functionality:**

```solidity
// VULNERABLE
function refund() public {
    uint256 amount = contributions[msg.sender];
    payable(msg.sender).transfer(amount); // External call first!
    contributions[msg.sender] = 0; // State update after
}

// SAFE
function refund() public {
    uint256 amount = contributions[msg.sender];
    contributions[msg.sender] = 0; // State update first!
    payable(msg.sender).transfer(amount); // External call after
}
```

### 2. Owner Withdrawal Before Sale Ends

**The problem:** Owner could withdraw funds before sale ends and disappear.

**The fix:**
```solidity
function finalizeSale() public {
    require(msg.sender == projectOwner);
    require(block.timestamp > saleEndTime, "Sale still active");
    finalized = true;
    payable(projectOwner).transfer(address(this).balance);
}
```

### 3. Integer Overflow in Token Calculation

**Solidity 0.8+** has automatic overflow protection, but be careful:

```solidity
// Could overflow if not careful
uint256 tokenAmount = (msg.value * 10**decimals) / tokenPrice;

// Safer with checks
require(msg.value <= type(uint256).max / 10**decimals, "Amount too large");
```

---

## Real-World ICO Patterns

### Dutch Auction (Gnosis)

**Price starts high, decreases over time:**
```solidity
function getCurrentPrice() public view returns (uint256) {
    uint256 elapsed = block.timestamp - saleStartTime;
    uint256 priceDecrease = (startPrice - endPrice) * elapsed / saleDuration;
    return startPrice - priceDecrease;
}
```

### Bonding Curve (Bancor)

**Price increases with supply:**
```solidity
function calculatePrice() public view returns (uint256) {
    // Price = supply^2
    return totalSupply * totalSupply / 1e18;
}
```

### Fair Launch (No Presale)

**Everyone buys at the same time:**
- No private sale
- No team allocation
- All tokens available at launch

---

## Key Concepts You've Learned

**1. Token sales** - ICO/presale mechanics

**2. Token locking** - Preventing early dumps with override

**3. Time-based logic** - Using block.timestamp for deadlines

**4. Inheritance override** - Modifying parent behavior

**5. Decimal handling** - Working with wei and token decimals

**6. Security patterns** - Reentrancy, access control

---

## Why This Matters

Token sales raised **$20+ billion** in 2017-2018:
- Funded thousands of projects
- Created new fundraising model
- Democratized investment
- Enabled global participation

Understanding token distribution is essential for launching any Web3 project.

## Challenge Yourself

Extend this token sale:
1. Add tiered pricing with early bird discounts
2. Implement a hard cap and soft cap
3. Create a whitelist for private sale
4. Add vesting schedules for team tokens
5. Build a refund mechanism if soft cap not met

You've mastered token distribution mechanics. This is real-world fundraising technology!
