# FortKnox: Understanding and Preventing Reentrancy Attacks

Welcome to one of the most important security lessons in smart contract development. Today, we're studying **reentrancy** - the vulnerability that caused the infamous **DAO hack** in 2016, resulting in the loss of $60 million and the split of Ethereum into ETH and ETC.

## The $60 Million Lesson

**June 17, 2016** - A hacker exploited a reentrancy vulnerability in The DAO smart contract and drained 3.6 million ETH (worth $60M at the time, $7+ billion at 2021 peak).

**The aftermath:**
- Ethereum community was forced to hard fork
- Created Ethereum (ETH) and Ethereum Classic (ETC)
- Changed smart contract security forever
- Made reentrancy the #1 vulnerability to understand

**This is not theoretical. This is real money. Real consequences.**

## What is Reentrancy?

Reentrancy is when a function can be called again **before the first call finishes**. In the context of smart contracts, it means:

1. Your contract calls an external contract (sending ETH)
2. That external contract calls back into your contract
3. Your contract's state hasn't been updated yet
4. The attacker can exploit this to drain funds

**Think of it like this:**

### The Vending Machine Analogy

Imagine a vending machine with a bug:

1. You insert $1 and press "Soda"
2. The machine dispenses the soda
3. **BEFORE** it deducts $1 from your credit, you press "Soda" again
4. It checks: "Do they have $1?" Yes! (hasn't been deducted yet)
5. Dispenses another soda
6. You keep pressing while it's dispensing
7. Free sodas forever!

**That's reentrancy.** The state (your credit) is updated AFTER the action (dispensing), creating a window for exploitation.

## The Vulnerable Contract

Let's build a vault that's vulnerable to reentrancy:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableVault {
    mapping(address => uint256) public balances;
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    // VULNERABLE - Don't use this!
    function vulnerableWithdraw() external {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        // 1. External call FIRST (sends ETH)
        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "Transfer failed");
        
        // 2. State update AFTER (too late!)
        balances[msg.sender] = 0;
    }
}
```

**What's wrong here?**

The order of operations:
1. ✅ Check balance
2. ❌ Send ETH (external call)
3. ❌ Update balance (state change)

**The problem:** Between steps 2 and 3, the attacker can call `vulnerableWithdraw()` again!

## The Attack Contract

Here's how an attacker exploits this:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IVulnerableVault {
    function deposit() external payable;
    function vulnerableWithdraw() external;
}

contract Attacker {
    IVulnerableVault public vault;
    uint256 public attackCount;
    
    constructor(address _vaultAddress) {
        vault = IVulnerableVault(_vaultAddress);
    }
    
    // Step 1: Deposit some ETH to have a balance
    function attack() external payable {
        require(msg.value >= 1 ether, "Need at least 1 ETH");
        vault.deposit{value: msg.value}();
        vault.vulnerableWithdraw();
    }
    
    // Step 2: This gets called when receiving ETH
    receive() external payable {
        attackCount++;
        
        // Keep withdrawing until vault is empty
        if (address(vault).balance >= 1 ether) {
            vault.vulnerableWithdraw();
        }
    }
    
    // Step 3: Attacker withdraws stolen funds
    function withdraw() external {
        payable(msg.sender).transfer(address(this).balance);
    }
}
```

**How the attack works:**

1. **Attacker deposits 1 ETH** into the vault
2. **Attacker calls `attack()`** which calls `vulnerableWithdraw()`
3. **Vault sends 1 ETH** to the attacker contract
4. **Attacker's `receive()` function is triggered**
5. **`receive()` calls `vulnerableWithdraw()` again** (reentrancy!)
6. **Vault checks balance** - Still 1 ETH! (hasn't been updated yet)
7. **Vault sends another 1 ETH**
8. **Loop continues** until vault is drained

**Timeline in milliseconds:**
```
0ms:  Attacker calls vulnerableWithdraw()
1ms:  Vault checks balance: 1 ETH ✓
2ms:  Vault sends 1 ETH to attacker
3ms:  Attacker's receive() is called
4ms:  Attacker calls vulnerableWithdraw() AGAIN
5ms:  Vault checks balance: STILL 1 ETH! ✓ (not updated yet)
6ms:  Vault sends another 1 ETH
7ms:  Loop continues...
```

## The Fix: Three Defense Patterns

### Defense #1: Checks-Effects-Interactions Pattern

**The golden rule:** Always update state BEFORE making external calls.

```solidity
function safeWithdraw() external {
    uint256 amount = balances[msg.sender];
    require(amount > 0, "No balance");
    
    // 1. CHECKS - Validate conditions
    // (already done with require)
    
    // 2. EFFECTS - Update state FIRST
    balances[msg.sender] = 0;
    
    // 3. INTERACTIONS - External calls LAST
    (bool sent, ) = msg.sender.call{value: amount}("");
    require(sent, "Transfer failed");
}
```

**Why this works:** When the attacker tries to re-enter, their balance is already 0. The `require(amount > 0)` check fails.

### Defense #2: Reentrancy Guard (Mutex Lock)

Use a lock to prevent reentrant calls:

```solidity
contract SecureVault {
    mapping(address => uint256) public balances;
    bool private locked;
    
    modifier nonReentrant() {
        require(!locked, "Reentrant call blocked");
        locked = true;
        _;
        locked = false;
    }
    
    function withdraw() external nonReentrant {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        balances[msg.sender] = 0;
        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "Transfer failed");
    }
}
```

**How it works:**
1. First call sets `locked = true`
2. Reentrant call hits `require(!locked)` and reverts
3. After first call completes, `locked = false`

**OpenZeppelin's ReentrancyGuard:**
```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyVault is ReentrancyGuard {
    function withdraw() external nonReentrant {
        // Your code here
    }
}
```

### Defense #3: Pull Over Push Pattern

Instead of sending ETH directly, let users withdraw it themselves:

```solidity
contract PullPaymentVault {
    mapping(address => uint256) public balances;
    mapping(address => uint256) public pendingWithdrawals;
    
    function initiateWithdraw() external {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        balances[msg.sender] = 0;
        pendingWithdrawals[msg.sender] += amount;
    }
    
    function withdraw() external {
        uint256 amount = pendingWithdrawals[msg.sender];
        require(amount > 0, "Nothing to withdraw");
        
        pendingWithdrawals[msg.sender] = 0;
        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "Transfer failed");
    }
}
```

**Advantage:** Separates state changes from ETH transfers, reducing attack surface.

## Real-World Examples of Reentrancy Attacks

### 1. The DAO Hack (2016) - $60M

**The vulnerability:**
```solidity
function splitDAO() {
    // ... code ...
    if (balances[msg.sender] > 0) {
        msg.sender.call.value(balances[msg.sender])(); // External call first
        balances[msg.sender] = 0; // State update after
    }
}
```

### 2. Lendf.Me (2020) - $25M

**The vulnerability:** Reentrancy in a DeFi lending protocol. Attacker borrowed, re-entered, borrowed again before collateral was updated.

### 3. Cream Finance (2021) - $130M

**The vulnerability:** Complex reentrancy involving flash loans and multiple contract interactions.

## Testing for Reentrancy

### Test Case: Vulnerable Contract

```javascript
describe("Reentrancy Attack", function() {
    it("Should drain the vault", async function() {
        // Deploy vulnerable vault
        const Vault = await ethers.getContractFactory("VulnerableVault");
        const vault = await Vault.deploy();
        
        // Add 10 ETH to vault from other users
        await vault.connect(user1).deposit({ value: ethers.utils.parseEther("5") });
        await vault.connect(user2).deposit({ value: ethers.utils.parseEther("5") });
        
        // Deploy attacker
        const Attacker = await ethers.getContractFactory("Attacker");
        const attacker = await Attacker.deploy(vault.address);
        
        // Execute attack with 1 ETH
        await attacker.attack({ value: ethers.utils.parseEther("1") });
        
        // Attacker should have drained the vault
        const attackerBalance = await ethers.provider.getBalance(attacker.address);
        expect(attackerBalance).to.be.gt(ethers.utils.parseEther("10"));
    });
});
```

### Test Case: Secure Contract

```javascript
it("Should prevent reentrancy", async function() {
    const SecureVault = await ethers.getContractFactory("SecureVault");
    const vault = await SecureVault.deploy();
    
    await vault.connect(user1).deposit({ value: ethers.utils.parseEther("5") });
    
    const Attacker = await ethers.getContractFactory("Attacker");
    const attacker = await Attacker.deploy(vault.address);
    
    // Attack should fail
    await expect(
        attacker.attack({ value: ethers.utils.parseEther("1") })
    ).to.be.reverted;
});
```

## Advanced Reentrancy: Cross-Function and Read-Only

### Cross-Function Reentrancy

```solidity
contract CrossFunctionVulnerable {
    mapping(address => uint256) public balances;
    
    function withdraw() external {
        uint256 amount = balances[msg.sender];
        (bool sent, ) = msg.sender.call{value: amount}("");
        balances[msg.sender] = 0;
    }
    
    function transfer(address to, uint256 amount) external {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

**The attack:** Call `withdraw()`, which triggers `receive()`, which calls `transfer()` before balance is updated.

### Read-Only Reentrancy

```solidity
contract PriceOracle {
    function getPrice() external view returns (uint256) {
        return token.balanceOf(address(this)) * pricePerToken;
    }
}

contract Vulnerable {
    function withdraw() external {
        uint256 amount = balances[msg.sender];
        (bool sent, ) = msg.sender.call{value: amount}("");
        balances[msg.sender] = 0;
    }
}
```

**The attack:** During `withdraw()`, attacker calls `getPrice()` which reads stale state, manipulating price oracles.

## Security Checklist

✅ **Use Checks-Effects-Interactions pattern**
✅ **Add reentrancy guards to all state-changing functions**
✅ **Prefer pull over push for payments**
✅ **Use OpenZeppelin's ReentrancyGuard**
✅ **Update state before external calls**
✅ **Test with attack contracts**
✅ **Get professional audits**

## Key Concepts You've Learned

**1. Reentrancy vulnerability** - Calling back before state updates

**2. Checks-Effects-Interactions** - The golden pattern for security

**3. Reentrancy guards** - Using locks to prevent reentrant calls

**4. Pull over push** - Letting users withdraw instead of sending

**5. Real-world impact** - The DAO hack and its consequences

**6. Testing attacks** - How to write exploit tests

## Why This Matters

Reentrancy is the **#1 vulnerability** in smart contracts. Understanding it is not optional - it's essential. Every major hack involves some form of reentrancy or state manipulation.

**Famous quotes:**
- "Code is law" - Until it's not (The DAO)
- "Don't trust, verify" - Especially external calls
- "Assume every external call is malicious" - Security 101

## Challenge Yourself

Practice secure coding:
1. Audit the vulnerable contract and find all attack vectors
2. Write a test suite that catches reentrancy
3. Implement a vault with multiple withdrawal methods (all secure)
4. Study the actual DAO hack code
5. Build a reentrancy detector tool

You've mastered the most critical security vulnerability in Web3. This knowledge will save millions!
