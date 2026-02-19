# Gas-Efficient Voting: Mastering Optimization Techniques

Welcome to one of the most important skills in Solidity development: **gas optimization**. Today, we're not just building a voting contract - we're building it to be as cheap as possible to run.

## The Gas Problem

**Reality check:** On Ethereum mainnet during high congestion:
- Simple token transfer: $5-20
- Uniswap swap: $50-200
- Complex DeFi interaction: $200-500
- NFT mint: $100-300

**Your users feel this pain.** Every extra operation, every wasted storage slot, every inefficient loop costs them real money.

## Why Gas Matters

**User experience:**
- High gas = fewer users
- High gas = smaller transactions
- High gas = people move to L2s or competitors

**Real example:** During the 2021 NFT boom, some mints cost $500+ in gas. Projects that optimized their contracts had a massive advantage.

**Your job as a developer:** Write code that works AND is cheap to run.

## What We're Building

A voting system optimized to the extreme using:
- **Storage packing** - Fitting multiple variables into single slots
- **Bit manipulation** - Using individual bits instead of booleans
- **Fixed-size types** - Avoiding dynamic arrays and strings
- **Efficient data structures** - Minimizing storage reads/writes

**The goal:** Make voting as cheap as possible while maintaining functionality.

---

## Understanding Gas Costs

### Storage Operations (The Expensive Ones)

**SSTORE (write to storage):**
- First time: 20,000 gas
- Updating existing: 5,000 gas
- Setting to zero (refund): -15,000 gas

**SLOAD (read from storage):**
- Cold access: 2,100 gas
- Warm access: 100 gas

**Memory operations:**
- Much cheaper: 3-6 gas per operation

**The lesson:** Minimize storage operations!

### Storage Slots

**Each storage slot is 32 bytes.** If you can pack multiple variables into one slot, you save massive gas.

**Example - Unoptimized:**
```solidity
uint8 a;    // Uses 1 full slot (32 bytes)
uint8 b;    // Uses another full slot (32 bytes)
uint8 c;    // Uses another full slot (32 bytes)
// Total: 3 slots = 60,000 gas to write all
```

**Example - Optimized:**
```solidity
uint8 a;    // Uses 1 byte of slot 1
uint8 b;    // Uses 1 byte of slot 1
uint8 c;    // Uses 1 byte of slot 1
// Total: 1 slot = 20,000 gas to write all
```

**Savings: 40,000 gas (67% reduction!)**

---

## Optimization Technique #1: Storage Packing

Let's see this in action with our voting contract:

### GasEfficientVoting.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract GasEfficientVoting {
    // Limits us to 255 proposals, but fits in a uint8 (1 byte)
    uint8 public proposalCount;
    
    struct Proposal {
        bytes32 name;       // 32 bytes
        uint32 voteCount;   // 4 bytes
        uint32 startTime;   // 4 bytes
        uint32 endTime;     // 4 bytes
        bool executed;      // 1 byte
        // Total: 32 + 4 + 4 + 4 + 1 = 45 bytes.
        // Actually, name is one slot. The rest (4+4+4+1 = 13 bytes) pack into a SECOND slot.
        // Total storage slots used: 2. (Unoptimized would be 3 or 4).
    }

    mapping(uint8 => Proposal) public proposals;
    
    // Each user has a "bitmap". 
    // If bit 0 is '1', they voted on proposal 0.
    // If bit 5 is '1', they voted on proposal 5.
    mapping(address => uint256) private voterRegistry;

    function createProposal(bytes32 _name) external {
        proposalCount++;
        uint32 currentTime = uint32(block.timestamp);
        
        proposals[proposalCount] = Proposal({
            name: _name,
            voteCount: 0,
            startTime: currentTime,
            endTime: currentTime + 1 days,
            executed: false
        });
    }

    function vote(uint8 proposalId) external {
        require(proposalId <= proposalCount && proposalId > 0, "Invalid ID");
        
        // 1. Create a "mask". 
        // If proposalId is 2, mask is ...00100 (binary)
        uint256 mask = 1 << proposalId;
        
        // 2. Check if they already voted using bitwise AND (&)
        // If (registry & mask) is NOT zero, the bit was already set.
        require((voterRegistry[msg.sender] & mask) == 0, "Already voted");
        
        // 3. Mark as voted using bitwise OR (|)
        // This flips the specific bit to 1, leaving others unchanged.
        voterRegistry[msg.sender] |= mask;
        
        // 4. Increment count
        proposals[proposalId].voteCount++;
    }

    function hasVoted(address voter, uint8 proposalId) external view returns (bool) {
        // Check the specific bit
        return (voterRegistry[voter] & (1 << proposalId)) != 0;
    }
}
```

**Breaking down the struct packing:**

```solidity
struct Proposal {
    bytes32 name;       // 32 bytes - Slot 0
    uint32 voteCount;   // 4 bytes  ┐
    uint32 startTime;   // 4 bytes  │ Slot 1 (13 bytes total)
    uint32 endTime;     // 4 bytes  │
    bool executed;      // 1 byte   ┘
}
```

**Storage layout:**
- **Slot 0:** `name` (32 bytes) - Full slot
- **Slot 1:** `voteCount` + `startTime` + `endTime` + `executed` (13 bytes) - Packed!

**Without packing:** Would use 5 slots (160 bytes)
**With packing:** Uses 2 slots (64 bytes)
**Savings:** 60% reduction in storage!

---

## Optimization Technique #2: Bit Manipulation

This is where things get really clever. Instead of storing a boolean for each vote:

```solidity
// EXPENSIVE - Each boolean uses a full storage slot
mapping(address => mapping(uint256 => bool)) public hasVoted;
```

We use **bitmaps** - a single `uint256` where each bit represents a vote:

```solidity
// CHEAP - One uint256 can track 256 votes
mapping(address => uint256) private voterRegistry;
```

### Understanding Bitmaps

A `uint256` is 256 bits. Each bit can be 0 or 1:

```
Binary: 00000000000000000000000000000101
         ↑                         ↑↑
         Bit 255                   Bit 2, 0 (voted on proposals 0 and 2)
```

**How we use it:**
- Bit 0 = 1 → Voted on proposal 0
- Bit 1 = 1 → Voted on proposal 1
- Bit 5 = 1 → Voted on proposal 5
- etc.

### The Voting Logic with Bit Manipulation

```solidity
function vote(uint8 proposalId) external {
    require(proposalId <= proposalCount && proposalId > 0, "Invalid ID");
    
    // 1. Create a "mask" for this proposal
    uint256 mask = 1 << proposalId;
    
    // 2. Check if they already voted using bitwise AND
    require((voterRegistry[msg.sender] & mask) == 0, "Already voted");
    
    // 3. Mark as voted using bitwise OR
    voterRegistry[msg.sender] |= mask;
    
    // 4. Increment count
    proposals[proposalId].voteCount++;
}
```

**Let's break down each operation:**

### Step 1: Creating the Mask

```solidity
uint256 mask = 1 << proposalId;
```

**What is `<<`?** Left shift operator. It moves bits to the left.

**Examples:**
- `1 << 0` = `0001` (binary) = 1 (decimal)
- `1 << 1` = `0010` (binary) = 2 (decimal)
- `1 << 2` = `0100` (binary) = 4 (decimal)
- `1 << 3` = `1000` (binary) = 8 (decimal)

**For proposal 5:**
```
1 << 5 = 00000000000000000000000000100000 (binary)
       = 32 (decimal)
```

This creates a number with only the 5th bit set to 1.

### Step 2: Checking if Already Voted

```solidity
require((voterRegistry[msg.sender] & mask) == 0, "Already voted");
```

**What is `&`?** Bitwise AND. Returns 1 only if both bits are 1.

**Example:**
```
Registry:  00000000000000000000000000100101 (voted on 0, 2, 5)
Mask:      00000000000000000000000000100000 (checking proposal 5)
AND:       00000000000000000000000000100000 (result is NOT zero!)
```

If the result is NOT zero, they already voted on this proposal.

**Another example (not voted yet):**
```
Registry:  00000000000000000000000000000101 (voted on 0, 2)
Mask:      00000000000000000000000000100000 (checking proposal 5)
AND:       00000000000000000000000000000000 (result IS zero!)
```

Result is zero = they haven't voted yet.

### Step 3: Marking as Voted

```solidity
voterRegistry[msg.sender] |= mask;
```

**What is `|=`?** Bitwise OR assignment. Sets a bit to 1.

**Example:**
```
Before:    00000000000000000000000000000101 (voted on 0, 2)
Mask:      00000000000000000000000000100000 (proposal 5)
OR:        00000000000000000000000000100101 (now voted on 0, 2, 5)
```

The 5th bit is now set to 1, marking proposal 5 as voted.

### Checking Vote Status

```solidity
function hasVoted(address voter, uint8 proposalId) external view returns (bool) {
    return (voterRegistry[voter] & (1 << proposalId)) != 0;
}
```

Same logic as the check - if AND result is not zero, they voted.

---

## Optimization Technique #3: Fixed-Size Types

### bytes32 vs string

```solidity
// EXPENSIVE - Dynamic string
string public name;  // Uses multiple storage slots + length tracking

// CHEAP - Fixed size
bytes32 public name; // Uses exactly 1 slot
```

**Trade-off:** `bytes32` can only hold 32 characters, but it's much cheaper.

**When to use each:**
- **bytes32** - Short, fixed data (names, symbols, IDs)
- **string** - Long, variable data (descriptions, URLs)

### uint32 vs uint256

```solidity
// Standard - Uses full 32 bytes
uint256 public timestamp;

// Optimized - Uses only 4 bytes
uint32 public timestamp;
```

**Why uint32 for timestamps?**
- `uint32` max value: 4,294,967,295 seconds
- That's year 2106
- Good enough for most use cases
- Saves 28 bytes per timestamp

**When to use smaller uints:**
- **uint8** - Counters (0-255)
- **uint16** - Larger counters (0-65,535)
- **uint32** - Timestamps, large counters
- **uint256** - Token amounts, large numbers

---

## Gas Cost Comparison

Let's compare our optimized contract vs a naive implementation:

### Naive Implementation

```solidity
contract NaiveVoting {
    struct Proposal {
        string name;                    // Dynamic, multiple slots
        uint256 voteCount;              // 32 bytes
        uint256 startTime;              // 32 bytes
        uint256 endTime;                // 32 bytes
        bool executed;                  // 32 bytes (not packed)
    }
    
    mapping(uint256 => Proposal) public proposals;
    mapping(address => mapping(uint256 => bool)) public hasVoted;
    
    function vote(uint256 proposalId) external {
        require(!hasVoted[msg.sender][proposalId]);
        hasVoted[msg.sender][proposalId] = true;
        proposals[proposalId].voteCount++;
    }
}
```

**Gas costs (approximate):**
- Create proposal: ~150,000 gas
- Vote: ~65,000 gas
- Check if voted: ~2,100 gas

### Optimized Implementation

```solidity
contract GasEfficientVoting {
    struct Proposal {
        bytes32 name;       // 32 bytes
        uint32 voteCount;   // 4 bytes
        uint32 startTime;   // 4 bytes
        uint32 endTime;     // 4 bytes
        bool executed;      // 1 byte
    }
    
    mapping(uint8 => Proposal) public proposals;
    mapping(address => uint256) private voterRegistry;
    
    function vote(uint8 proposalId) external {
        uint256 mask = 1 << proposalId;
        require((voterRegistry[msg.sender] & mask) == 0);
        voterRegistry[msg.sender] |= mask;
        proposals[proposalId].voteCount++;
    }
}
```

**Gas costs (approximate):**
- Create proposal: ~80,000 gas (47% cheaper!)
- Vote: ~45,000 gas (31% cheaper!)
- Check if voted: ~400 gas (81% cheaper!)

**For 1000 votes:** Saves ~20,000,000 gas = ~$2,000-10,000 depending on gas prices!

---

## Advanced Optimization Techniques

### 1. Caching Storage Variables

```solidity
// BAD - Multiple storage reads
function badExample() external {
    proposals[1].voteCount++;  // SLOAD
    proposals[1].voteCount++;  // SLOAD again
    proposals[1].voteCount++;  // SLOAD again
}

// GOOD - Cache in memory
function goodExample() external {
    Proposal storage prop = proposals[1];  // One SLOAD
    prop.voteCount++;
    prop.voteCount++;
    prop.voteCount++;
}
```

### 2. Short-Circuit Evaluation

```solidity
// BAD - Always checks both conditions
require(expensiveCheck() && cheapCheck());

// GOOD - Stops at first false
require(cheapCheck() && expensiveCheck());
```

### 3. Unchecked Math (When Safe)

```solidity
// With overflow checks (post 0.8.0)
for (uint256 i = 0; i < 100; i++) {
    // Compiler adds overflow check on i++
}

// Without checks (when you know it's safe)
for (uint256 i = 0; i < 100;) {
    // Your code
    unchecked { i++; }  // Saves ~30 gas per iteration
}
```

### 4. Calldata vs Memory

```solidity
// EXPENSIVE - Copies to memory
function vote(uint256[] memory proposalIds) external {
    // ...
}

// CHEAP - Reads directly from calldata
function vote(uint256[] calldata proposalIds) external {
    // ...
}
```

---

## Real-World Gas Optimization Examples

### Uniswap V3

Uniswap V3 uses extreme optimization:
- Bit-packed tick data
- Custom assembly for critical paths
- Minimized storage operations
- Result: 30% cheaper swaps than V2

### ERC721A (Azuki)

Optimized NFT minting:
- Batch minting in single transaction
- Lazy initialization of token data
- Result: Mint 5 NFTs for the price of 1

### Solady Library

Ultra-optimized implementations:
- Custom assembly for common operations
- Bit manipulation for flags
- Result: 10-50% gas savings vs standard libraries

---

## Gas Optimization Checklist

✅ **Pack structs** - Order variables by size
✅ **Use bitmaps** - For boolean flags
✅ **Fixed-size types** - bytes32 over string when possible
✅ **Smaller uints** - uint8/uint32 when appropriate
✅ **Cache storage** - Read once, use multiple times
✅ **Use calldata** - For read-only array parameters
✅ **Avoid loops** - Or use unchecked when safe
✅ **Short-circuit** - Put cheap checks first
✅ **Immutable/constant** - For values that don't change

---

## Key Concepts You've Learned

**1. Storage packing** - Fitting multiple variables in one slot

**2. Bit manipulation** - Using individual bits for flags

**3. Fixed-size types** - bytes32 and smaller uints

**4. Gas cost awareness** - Understanding SSTORE/SLOAD costs

**5. Optimization trade-offs** - Complexity vs savings

**6. Real-world impact** - How optimization affects user experience

---

## Why This Matters

Gas optimization is what separates good developers from great ones. In production:
- **Users care** - High gas = bad UX
- **Protocols care** - Efficiency = competitive advantage
- **Auditors care** - Optimization = professionalism

**Famous quote:** "Premature optimization is the root of all evil" - BUT in Solidity, gas optimization is not premature. It's essential.

## Challenge Yourself

Practice optimization:
1. Optimize an ERC20 token contract
2. Build a gas-efficient NFT minting system
3. Create a bitmap-based permission system
4. Audit a contract and find optimization opportunities
5. Compare gas costs before/after optimization

You've mastered gas optimization. This skill will save your users millions in fees!

---

### Why is this faster?
*   **Fewer SSTORE operations:** Updating a bitmap is often cheaper than initializing a new storage slot for a mapping.
*   **Smaller Data:** Loading `uint32` is sometimes cheaper when reading structs, as one `SLOAD` instruction can fetch multiple fields if they are packed into one slot.

Mastering these tricks separates the juniors from the seniors!
