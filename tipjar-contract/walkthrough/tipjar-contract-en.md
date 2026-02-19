# TipJar Contract: Multi-Currency Tipping System

Welcome to a practical real-world application: **accepting payments in multiple currencies**. Today, you're building a tipping system that accepts USD, EUR, JPY, and more - all converted to ETH automatically.

## The Global Payment Problem

**The scenario:**
- You're a content creator accepting crypto tips
- Fans from USA, Europe, Japan want to tip
- They think in their local currency (USD, EUR, JPY)
- But you receive ETH

**The challenge:**
- "How much is $5 in ETH?"
- "What's 10 EUR worth?"
- Exchange rates change constantly

**The solution:** A smart contract that handles conversion automatically.

## What We're Building

A multi-currency tip jar with:
- **Currency support** - USD, EUR, JPY, GBP, and more
- **Automatic conversion** - Calculate ETH equivalent
- **Flexible tipping** - Users choose their currency
- **Owner withdrawal** - Collect all tips
- **Contribution tracking** - See who tipped what

**Real-world use cases:**
- Content creator tips
- Donation platforms
- Crowdfunding
- International payments
- Freelancer invoicing

### What Data Do We Need for Our TipJar?

```
address public owner;
mapping(string => uint256) public conversionRates;
string[] public supportedCurrencies;
uint256 public totalTipsReceived;
mapping(address => uint256) public tipperContributions;
mapping(string => uint256) public tipsPerCurrency;
```

### Modifiers

```
modifier onlyOwner() {
    require(msg.sender == owner, "Only owner can perform this action");
    _;
}
```

### Setting Up Currency Conversion — addCurrency()

```
function addCurrency(string memory _currencyCode, uint256 _rateToEth) public onlyOwner {
    require(_rateToEth > 0, "Conversion rate must be greater than 0");

    bool currencyExists = false;
    for (uint i = 0; i < supportedCurrencies.length; i++) {
        if (keccak256(bytes(supportedCurrencies[i])) == keccak256(bytes(_currencyCode))) {
            currencyExists = true;
            break;
        }
    }

    if (!currencyExists) {
        supportedCurrencies.push(_currencyCode);
    }

    conversionRates[_currencyCode] = _rateToEth;
}
```

### The Constructor

```
constructor() {
    owner = msg.sender;

    addCurrency("USD", 5 * 10**14);
    addCurrency("EUR", 6 * 10**14);
    addCurrency("JPY", 4 * 10**12);
    addCurrency("GBP", 7 * 10**14);
}
```

### Converting to ETH (in wei)

```
function convertToEth(string memory _currencyCode, uint256 _amount) public view returns (uint256) {
    require(conversionRates[_currencyCode] > 0, "Currency not supported");

    uint256 ethAmount = _amount * conversionRates[_currencyCode];
    return ethAmount;
}
```

### Sending a Tip in ETH

```
function tipInEth() public payable {
    require(msg.value > 0, "Tip amount must be greater than 0");

    tipperContributions[msg.sender] += msg.value;
    totalTipsReceived += msg.value;
    tipsPerCurrency["ETH"] += msg.value;
}
```

### Tipping in a Foreign Currency

```
function tipInCurrency(string memory _currencyCode, uint256 _amount) public payable {
    require(conversionRates[_currencyCode] > 0, "Currency not supported");
    require(_amount > 0, "Amount must be greater than 0");

    uint256 ethAmount = convertToEth(_currencyCode, _amount);
    require(msg.value == ethAmount, "Sent ETH doesn't match the converted amount");

    tipperContributions[msg.sender] += msg.value;
    totalTipsReceived += msg.value;
    tipsPerCurrency[_currencyCode] += _amount;
}
```

### Withdrawing Tips

```
function withdrawTips() public onlyOwner {
    uint256 contractBalance = address(this).balance;
    require(contractBalance > 0, "No tips to withdraw");

    (bool success, ) = payable(owner).call{value: contractBalance}("");
    require(success, "Transfer failed");

    totalTipsReceived = 0;
}
```

### Utility Functions

```
function getSupportedCurrencies() public view returns (string[] memory) {
    return supportedCurrencies;
}

function getContractBalance() public view returns (uint256) {
    return address(this).balance;
}

function getTipperContribution(address _tipper) public view returns (uint256) {
    return tipperContributions[_tipper];
}

function getTipsInCurrency(string memory _currencyCode) public view returns (uint256) {
    return tipsPerCurrency[_currencyCode];
}

function getConversionRate(string memory _currencyCode) public view returns (uint256) {
    require(conversionRates[_currencyCode] > 0, "Currency not supported");
    return conversionRates[_currencyCode];
}
```

---

## Understanding Currency Conversion

### How Conversion Rates Work

```solidity
conversionRates["USD"] = 5 * 10**14; // 0.0005 ETH per USD
```

**What this means:**
- 1 USD = 0.0005 ETH
- $2,000 = 1 ETH
- If ETH = $2,000, this is accurate

**The math:**
```solidity
function convertToEth(string memory _currencyCode, uint256 _amount) public view returns (uint256) {
    uint256 ethAmount = _amount * conversionRates[_currencyCode];
    return ethAmount;
}

// Example: $10 tip
// 10 * (5 * 10**14) = 5 * 10**15 wei = 0.005 ETH
```

### Why Wei?

**ETH denominations:**
- 1 ETH = 1,000,000,000,000,000,000 wei (10^18)
- 1 Gwei = 1,000,000,000 wei (10^9)
- 1 wei = smallest unit

**Why use wei?** Solidity doesn't support decimals, so we work in the smallest unit.

---

## String Comparison in Solidity

### The Problem

```solidity
// This doesn't work!
if (currency == "USD") { }
```

**Solidity can't compare strings directly.**

### The Solution: keccak256

```solidity
if (keccak256(bytes(supportedCurrencies[i])) == keccak256(bytes(_currencyCode))) {
    currencyExists = true;
}
```

**What's happening:**
1. Convert strings to bytes
2. Hash both with keccak256
3. Compare the hashes

**Why?** Hashes are fixed-size (32 bytes) and cheap to compare.

---

## Key Concepts You've Learned

**1. Multi-currency support** - Accepting different currencies

**2. Currency conversion** - Calculating ETH equivalents

**3. String comparison** - Using keccak256 for strings

**4. Wei calculations** - Working with smallest units

**5. Contribution tracking** - Recording who tipped

**6. Owner withdrawal** - Collecting accumulated tips

---

## Real-World Integration: Oracles

### The Problem with Fixed Rates

**Our current approach:**
```solidity
conversionRates["USD"] = 5 * 10**14; // Fixed rate
```

**The problem:** ETH price changes constantly!
- Today: 1 ETH = $2,000
- Tomorrow: 1 ETH = $1,800
- Fixed rate becomes inaccurate

### The Solution: Chainlink Price Feeds

```solidity
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract TipJarWithOracle {
    AggregatorV3Interface internal priceFeed;
    
    constructor() {
        // ETH/USD price feed
        priceFeed = AggregatorV3Interface(0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419);
    }
    
    function getLatestPrice() public view returns (uint256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();
        return uint256(price); // Price in USD with 8 decimals
    }
    
    function convertToEth(uint256 _usdAmount) public view returns (uint256) {
        uint256 ethPrice = getLatestPrice(); // e.g., 200000000000 ($2,000)
        return (_usdAmount * 1e18) / ethPrice;
    }
}
```

**Now rates update automatically!**

---

## Security Considerations

### 1. Rate Manipulation

**Problem:** Owner could set unfair rates.

**Solution:**
```solidity
modifier reasonableRate(uint256 _rate) {
    require(_rate > 1e12 && _rate < 1e16, "Rate out of bounds");
    _;
}

function addCurrency(string memory _currencyCode, uint256 _rateToEth) 
    public onlyOwner reasonableRate(_rateToEth) 
{
    // ...
}
```

### 2. Withdrawal Reentrancy

**Current code is safe:**
```solidity
function withdrawTips() public onlyOwner {
    uint256 contractBalance = address(this).balance;
    totalTipsReceived = 0; // State update before transfer
    
    (bool success, ) = payable(owner).call{value: contractBalance}("");
    require(success);
}
```

### 3. Integer Overflow

**Be careful with multiplication:**
```solidity
uint256 ethAmount = _amount * conversionRates[_currencyCode];
// Could overflow if _amount is huge
```

**Solidity 0.8+** protects automatically, but validate inputs.

---

## Real-World Tipping Platforms

### Brave Browser Tips

**How it works:**
- Users tip content creators in BAT tokens
- Automatic conversion from fiat
- Creators withdraw to bank or crypto

### Gitcoin Grants

**Quadratic funding:**
- Accept donations in multiple tokens
- Convert to common denomination
- Match funding based on contributions

### Crypto Payment Processors

**BitPay, Coinbase Commerce:**
- Accept crypto payments
- Display prices in local currency
- Instant conversion

---

## Why This Matters

Multi-currency support enables:
- **Global accessibility** - Users pay in familiar currency
- **Better UX** - No mental math required
- **Wider adoption** - Lower barrier to entry
- **Flexibility** - Support any currency

This pattern is used in every crypto payment system.

## Challenge Yourself

Extend this tip jar:
1. Integrate Chainlink price feeds for live rates
2. Add support for multiple cryptocurrencies (BTC, USDC)
3. Implement tipping goals with progress tracking
4. Create a leaderboard of top tippers
5. Add recurring tips (subscriptions)

You've mastered multi-currency payment systems. This is real-world Web3 development!
