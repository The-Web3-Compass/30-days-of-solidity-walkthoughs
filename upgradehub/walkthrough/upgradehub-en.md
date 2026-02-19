# UpgradeHub: Building Upgradeable Smart Contracts

Welcome to one of the most critical patterns in production Web3: **upgradeable contracts**. Today, you're learning how to build systems that can evolve without losing state or user data - the technology behind every major DeFi protocol.

## The Immutability Problem

**The scenario:**
- You deploy a contract with 10,000 users
- $10M in TVL (Total Value Locked)
- Then you discover a critical bug
- Or want to add a new feature
- Or need to optimize gas costs

**In Web2:**
```bash
git push origin main
# Deploy updated code
# Users get new features automatically
```

**In Web3:**
```solidity
// Contract deployed at 0x123...
// Immutable forever
// Can't change code
// Can't fix bugs
// Users stuck with old version
```

**The problem:** Smart contracts are immutable by design. Once deployed, the code is permanent.

**Real-world disasters:**
- **Parity Wallet bug** - $300M frozen forever
- **DAO hack** - $60M stolen, couldn't fix
- **Early DeFi bugs** - Millions lost

**The solution:** Proxy patterns that separate storage from logic.

## What Does Upgradeable Mean?

Upgradeable contracts separate storage from logic:
- One contract stores the data (the proxy).
- Another contract holds the logic.
- The proxy uses `delegatecall` to execute logic from the logic contract on its own storage. If you need to change behavior, you point the proxy to a new logic contract.

## What We're Building: An Upgradeable Subscription System

We'll build a modular subscription manager with:
1. `SubscriptionStorageLayout.sol`: Shared blueprint for variable order.
2. `SubscriptionStorage.sol`: The proxy contract handling `delegatecall`.
3. `SubscriptionLogicV1.sol`: Initial logic for plans and subscriptions.
4. `SubscriptionLogicV2.sol`: Upgraded logic with pause/resume functions.

```solidity
// SubscriptionStorageLayout.sol
contract SubscriptionStorageLayout {
    address public logicContract;
    address public owner;
    struct Subscription {
        uint8 planId;
        uint256 expiry;
        bool paused;
    }
    mapping(address => Subscription) public subscriptions;
    mapping(uint8 => uint256) public planPrices;
    mapping(uint8 => uint256) public planDuration;
}
```

```solidity
// SubscriptionStorage.sol (Proxy)
contract SubscriptionStorage is SubscriptionStorageLayout {
    constructor(address _logicContract) {
        owner = msg.sender;
        logicContract = _logicContract;
    }
    function upgradeTo(address _newLogic) external {
        require(msg.sender == owner, "Not owner");
        logicContract = _newLogic;
    }
    fallback() external payable {
        address impl = logicContract;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}

---

## Understanding Proxy Patterns

### How Upgradeability Works

**The architecture:**
```
User → Proxy Contract (Storage) → Logic Contract (Code)
        ↓ delegatecall
        Executes logic in proxy's context
```

**Key insight:** delegatecall executes another contract's code but uses YOUR storage.

### The Delegatecall Magic

```solidity
fallback() external payable {
    address impl = logicContract;
    assembly {
        // Copy call data
        calldatacopy(0, 0, calldatasize())
        
        // Execute logic contract's code in proxy's context
        let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
        
        // Copy return data
        returndatacopy(0, 0, returndatasize())
        
        // Return or revert
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```

**What's happening:**
1. User calls proxy
2. Proxy forwards to logic contract
3. Logic executes in proxy's storage
4. State changes saved in proxy
5. Logic can be swapped anytime

---

## Storage Layout: The Critical Rule

### Why Storage Layout Matters

**The danger:**
```solidity
// V1
contract LogicV1 {
    address public owner;      // Slot 0
    uint256 public value;      // Slot 1
}

// V2 - WRONG!
contract LogicV2 {
    uint256 public value;      // Slot 0 (was owner!)
    address public owner;      // Slot 1 (was value!)
}
```

**Result:** Data corruption! Owner becomes a number, value becomes an address.

### The Solution: Storage Layout Contract

```solidity
contract StorageLayout {
    address public logicContract;  // Slot 0
    address public owner;          // Slot 1
    // ... all storage variables
}

contract LogicV1 is StorageLayout {
    // Inherits storage layout
    // Can add logic safely
}

contract LogicV2 is StorageLayout {
    // Same storage layout
    // Different logic
}
```

**Rule:** Never change the order of existing variables. Only append new ones.

---

## Key Concepts You've Learned

**1. Proxy pattern** - Separating storage from logic

**2. Delegatecall** - Executing code in different context

**3. Storage layout** - Critical for upgrades

**4. Fallback function** - Forwarding calls

**5. Assembly** - Low-level EVM operations

**6. Upgradeability** - Evolving contracts safely

---

## Real-World Proxy Patterns

### 1. Transparent Proxy (OpenZeppelin)

```solidity
contract TransparentUpgradeableProxy {
    address private _admin;
    address private _implementation;
    
    modifier ifAdmin() {
        if (msg.sender == _admin) {
            _;
        } else {
            _fallback();
        }
    }
    
    function upgradeTo(address newImplementation) external ifAdmin {
        _implementation = newImplementation;
    }
}
```

**Feature:** Admin calls don't forward to implementation.

### 2. UUPS (Universal Upgradeable Proxy Standard)

```solidity
contract UUPSUpgradeable {
    function upgradeTo(address newImplementation) external {
        // Upgrade logic in implementation, not proxy
        _authorizeUpgrade(newImplementation);
        _implementation = newImplementation;
    }
}
```

**Feature:** Upgrade logic in implementation (cheaper proxy).

### 3. Beacon Proxy

```solidity
contract BeaconProxy {
    address private immutable _beacon;
    
    function implementation() public view returns (address) {
        return IBeacon(_beacon).implementation();
    }
}
```

**Feature:** Multiple proxies share one beacon (upgrade all at once).

---

## Security Considerations

### 1. Storage Collision

**Problem:** Proxy and logic use same storage slots.

**Solution:** Use unique storage slots (EIP-1967).
```solidity
bytes32 private constant IMPLEMENTATION_SLOT = 
    bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1);
```

### 2. Uninitialized Proxies

**Problem:** Constructor doesn't run for proxies.

**Solution:** Use initializer functions.
```solidity
contract LogicV1 {
    bool private initialized;
    
    function initialize(address _owner) external {
        require(!initialized, "Already initialized");
        owner = _owner;
        initialized = true;
    }
}
```

### 3. Selfdestruct in Logic

**Problem:** Logic contract selfdestructs, proxy breaks.

**Solution:** Never use selfdestruct in upgradeable contracts.

### 4. Delegatecall to Untrusted Contracts

**Problem:** Malicious logic can steal funds.

**Solution:** Only admin can upgrade, audit new logic.

---

## Major Protocols Using Proxies

### Aave V3

**Architecture:**
- LendingPool (proxy)
- Multiple logic contracts
- Upgradeable modules

**Why:** Fix bugs, add features, optimize gas.

### Compound

**Architecture:**
- Comptroller (proxy)
- CToken implementations
- Governance-controlled upgrades

### Uniswap V3

**Architecture:**
- Factory (immutable)
- Pools (upgradeable via governance)

---

## Why This Matters

Upgradeability enables:
- **Bug fixes** - Save millions from exploits
- **Feature additions** - Evolve with market
- **Gas optimization** - Reduce costs over time
- **Regulatory compliance** - Adapt to laws
- **Competitive advantage** - Iterate faster

**$100B+ in DeFi** relies on upgradeable contracts.

## Challenge Yourself

Extend this upgradeable system:
1. Add access control with multi-sig for upgrades
2. Implement timelock for upgrade delays
3. Create a voting system for upgrade proposals
4. Add emergency pause functionality
5. Build a version history tracker

You've mastered upgradeable contracts. This is production-level Web3 architecture!
