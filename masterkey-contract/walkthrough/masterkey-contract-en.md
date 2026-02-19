# Masterkey Contract: Mastering Inheritance and Code Reusability

Welcome to a game-changing concept in Solidity: **inheritance**. Up until now, you've been writing contracts one at a time, copying and pasting the same code over and over. Today, that changes.

## The Copy-Paste Problem

You've probably written something like this multiple times:

```solidity
contract VaultA {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    // ... vault logic ...
}

contract VaultB {
    address public owner;  // Copied again!
    
    constructor() {
        owner = msg.sender;  // Copied again!
    }
    
    modifier onlyOwner() {  // Copied again!
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    // ... different vault logic ...
}
```

**This is painful:**
- Duplicate code everywhere
- If you find a bug, fix it in 10 places
- Hard to maintain
- Easy to make mistakes

**There's a better way: inheritance.**

#### What Is Inheritance?

Let’s say your parent owns a house. One day, they pass it down to you. Now when you inherit that house, you're not just getting the four walls — you're getting everything that comes with it: the furniture, the rules, maybe even the family dog. You didn’t set any of that up yourself — but you still have access to all of it. That’s exactly what inheritance is like in Solidity.


#### Why Use It?

* Avoid writing the same logic in multiple places
* Split your code into smaller, focused pieces
* Reuse important features like access control or utility functions
* Make your contracts easier to update and maintain

#### What If You Want to Change Something?

Solidity gives you a safe way to do that using two keywords:

> virtual goes in the parent contract. It marks a function as changeable.

> override goes in the child contract. It tells Solidity, “I know this function was inherited, but I’m replacing it with my own version.”

#### What We’re Building Today

* Ownable.sol – A simple contract that defines who the owner is and gives you a reusable onlyOwner check.
* VaultMaster.sol – A vault that accepts ETH deposits, but only lets the owner withdraw, inheriting from Ownable.

### Contract 1: Ownable.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Ownable {
    address private owner;
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    constructor() {
        owner = msg.sender;
        emit OwnershipTransferred(address(0), msg.sender);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can perform this action");
        _;
    }

    function ownerAddress() public view returns (address) {
        return owner;
    }

    function transferOwnership(address _newOwner) public onlyOwner {
        require(_newOwner != address(0), "Invalid address");
        address previous = owner;
        owner = _newOwner;
        emit OwnershipTransferred(previous, _newOwner);
    }
}
```

### Contract 2: VaultMaster.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;
import "./ownable.sol";

contract VaultMaster is Ownable {
    event DepositSuccessful(address indexed account, uint256 value);
    event WithdrawSuccessful(address indexed recipient, uint256 value);

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }

    function deposit() public payable {
        require(msg.value > 0, "Enter a valid amount");
        emit DepositSuccessful(msg.sender, msg.value);
    }

    function withdraw(address _to, uint256 _amount) public onlyOwner {
        require(_amount <= getBalance(), "Insufficient balance");
        (bool success, ) = payable(_to).call{value: _amount}("");
        require(success, "Transfer Failed");
        emit WithdrawSuccessful(_to, _amount);
    }
}
```

---

## Understanding Inheritance Patterns

### Single Inheritance

```solidity
contract Parent {
    uint256 public value;
    
    function setValue(uint256 _value) public {
        value = _value;
    }
}

contract Child is Parent {
    // Child automatically has 'value' and 'setValue'
    // Can add new functionality
    function doubleValue() public {
        value = value * 2;
    }
}
```

**The `is` keyword** establishes the inheritance relationship. `Child is Parent` means "Child inherits from Parent."

### The Virtual and Override Keywords

```solidity
contract Parent {
    function greet() public virtual returns (string memory) {
        return "Hello from Parent";
    }
}

contract Child is Parent {
    function greet() public override returns (string memory) {
        return "Hello from Child";
    }
}
```

**Key concepts:**
- **`virtual`** - Marks a function as "changeable" in child contracts
- **`override`** - Indicates you're replacing the parent's implementation

**Without these keywords:** Solidity won't let you override functions (safety feature).

### Multiple Inheritance

```solidity
contract Ownable {
    address public owner;
    modifier onlyOwner() { require(msg.sender == owner); _; }
}

contract Pausable {
    bool public paused;
    modifier whenNotPaused() { require(!paused); _; }
}

contract MyContract is Ownable, Pausable {
    // Inherits from BOTH contracts
    function doSomething() public onlyOwner whenNotPaused {
        // Can use modifiers from both parents
    }
}
```

**Order matters!** If both parents have the same function, the rightmost parent wins.

---

## Real-World Inheritance Examples

### OpenZeppelin's Ownable

The industry-standard implementation:

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

contract VaultMaster is Ownable {
    constructor() Ownable(msg.sender) {}
    
    function withdraw(address _to, uint256 _amount) public onlyOwner {
        // Your logic here
    }
}
```

**What you get:**
- `owner()` - Returns current owner
- `onlyOwner` modifier - Access control
- `transferOwnership()` - Transfer to new owner
- `renounceOwnership()` - Remove owner (irreversible!)

### OpenZeppelin's ReentrancyGuard

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyVault is ReentrancyGuard {
    function withdraw() public nonReentrant {
        // Protected from reentrancy attacks
    }
}
```

### OpenZeppelin's Pausable

```solidity
import "@openzeppelin/contracts/security/Pausable.sol";

contract MyToken is Pausable {
    function transfer() public whenNotPaused {
        // Can be paused in emergencies
    }
}
```

---

## Advanced Inheritance Concepts

### Super Keyword

```solidity
contract Parent {
    function greet() public virtual returns (string memory) {
        return "Hello";
    }
}

contract Child is Parent {
    function greet() public override returns (string memory) {
        string memory parentGreeting = super.greet();
        return string(abi.encodePacked(parentGreeting, " from Child"));
    }
}
```

**`super`** calls the parent's version of the function. Result: "Hello from Child"

### Abstract Contracts

```solidity
abstract contract Animal {
    function makeSound() public virtual returns (string memory);
    // No implementation - child MUST implement this
}

contract Dog is Animal {
    function makeSound() public override returns (string memory) {
        return "Woof!";
    }
}
```

**Abstract contracts** define interfaces that children must implement.

### Interfaces

```solidity
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract MyToken is IERC20 {
    // Must implement all interface functions
    function transfer(address to, uint256 amount) external returns (bool) {
        // Implementation
    }
    
    function balanceOf(address account) external view returns (uint256) {
        // Implementation
    }
}
```

**Interfaces** are like abstract contracts but stricter - no implementation allowed.

---

## Inheritance Best Practices

### 1. Keep Parent Contracts Focused

```solidity
// GOOD - Single responsibility
contract Ownable {
    // Only ownership logic
}

contract Pausable {
    // Only pause logic
}

// BAD - Too much in one contract
contract OwnerPausableTimelockGovernance {
    // Too many responsibilities
}
```

### 2. Use Virtual Sparingly

```solidity
// GOOD - Only mark functions that should be overrideable
contract Base {
    function criticalLogic() public {
        // Not virtual - can't be changed
    }
    
    function customizableLogic() public virtual {
        // Virtual - can be customized
    }
}
```

### 3. Document Inheritance Chains

```solidity
/**
 * @dev MyVault inherits from:
 * - Ownable: Access control
 * - ReentrancyGuard: Security
 * - Pausable: Emergency stops
 */
contract MyVault is Ownable, ReentrancyGuard, Pausable {
    // ...
}
```

---

## Common Inheritance Patterns

### Pattern 1: Access Control Hierarchy

```solidity
contract Roles {
    mapping(address => bool) public admins;
    mapping(address => bool) public moderators;
}

contract MyContract is Roles {
    modifier onlyAdmin() {
        require(admins[msg.sender], "Not admin");
        _;
    }
    
    modifier onlyModerator() {
        require(moderators[msg.sender], "Not moderator");
        _;
    }
}
```

### Pattern 2: Upgradeable Contracts

```solidity
contract Storage {
    uint256 public value;
}

contract LogicV1 is Storage {
    function setValue(uint256 _value) public {
        value = _value;
    }
}

contract LogicV2 is Storage {
    function setValue(uint256 _value) public {
        value = _value * 2; // New logic
    }
}
```

### Pattern 3: Feature Modules

```solidity
contract Withdrawable {
    function withdraw() public virtual {
        // Base withdraw logic
    }
}

contract TimelockWithdrawable is Withdrawable {
    uint256 public unlockTime;
    
    function withdraw() public override {
        require(block.timestamp >= unlockTime, "Locked");
        super.withdraw();
    }
}
```

---

## Inheritance vs Composition

**Inheritance (is-a relationship):**
```solidity
contract Dog is Animal {
    // Dog IS AN Animal
}
```

**Composition (has-a relationship):**
```solidity
contract Wallet {
    IERC20 public token; // Wallet HAS A token
    
    constructor(address _token) {
        token = IERC20(_token);
    }
}
```

**When to use each:**
- **Inheritance:** Shared behavior, code reuse
- **Composition:** Flexibility, loose coupling

---

## Key Concepts You've Learned

**1. Inheritance** - Reusing code across contracts

**2. Virtual/Override** - Customizing parent functionality

**3. Multiple inheritance** - Combining multiple parents

**4. Super keyword** - Calling parent implementations

**5. Abstract contracts** - Defining interfaces

**6. OpenZeppelin patterns** - Industry-standard implementations

---

## Why This Matters

Inheritance is how professional developers build maintainable smart contracts:
- **OpenZeppelin** - All their contracts use inheritance
- **Aave** - Modular architecture with inheritance
- **Uniswap** - Base contracts inherited by pools
- **Every major protocol** - Uses inheritance patterns

Without inheritance, codebases become unmaintainable nightmares.

## Challenge Yourself

Practice inheritance:
1. Create a role-based access control system with multiple levels
2. Build an upgradeable contract using inheritance
3. Implement a feature flag system with inheritance
4. Create your own reusable base contracts
5. Study OpenZeppelin's inheritance patterns

You've mastered code reusability. This is professional-level Solidity development!
