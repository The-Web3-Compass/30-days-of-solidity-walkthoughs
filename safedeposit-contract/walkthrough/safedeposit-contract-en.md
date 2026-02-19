# SafeDeposit Contract: Mastering Interfaces and Abstract Contracts

Welcome to advanced Solidity architecture! Today, you're learning **interfaces and abstract contracts** - the building blocks of modular, extensible smart contract systems.

## The Problem: Rigid Contract Design

**Imagine building a vault system:**
- BasicVault - Simple storage
- PremiumVault - With metadata
- TimeLockedVault - With time delays
- MultiSigVault - With multiple owners

**The naive approach:**
```solidity
contract VaultManager {
    function createBasicVault() public { /* ... */ }
    function createPremiumVault() public { /* ... */ }
    function createTimeLockedVault() public { /* ... */ }
    function createMultiSigVault() public { /* ... */ }
    // Add new vault type? Modify VaultManager!
}
```

**Problems:**
- Tightly coupled
- Can't add new vault types without modifying manager
- Duplicate code everywhere
- Hard to test and maintain

**The solution:** Interfaces + Abstract Contracts + Polymorphism

### IDepositBox.sol (Interface)

```solidity
interface IDepositBox {
    function getOwner() external view returns (address);
    function transferOwnership(address newOwner) external;
    function storeSecret(string calldata secret) external;
    function getSecret() external view returns (string memory);
    function getBoxType() external pure returns (string memory);
    function getDepositTime() external view returns (uint256);
}
```

### BaseDepositBox.sol (Abstract Contract)

```solidity
abstract contract BaseDepositBox is IDepositBox {
    address private owner;
    string private secret;
    uint256 private depositTime;

    modifier onlyOwner() { require(msg.sender == owner, "Not owner"); _; }

    constructor() { owner = msg.sender; depositTime = block.timestamp; }

    function storeSecret(string calldata _secret) external virtual override onlyOwner {
        secret = _secret;
    }

    function getSecret() public view virtual override onlyOwner returns (string memory) {
        return secret;
    }

    function getOwner() public view override returns (address) { return owner; }
    function getDepositTime() external view virtual override returns (uint256) { return depositTime; }
}
```

### Specific Box Implementations

* BasicDepositBox: Returns "Basic" as box type.
* PremiumDepositBox: Adds metadata storage features.
* TimeLockedDepositBox: Adds `timeUnlocked` modifier to `getSecret`.

---

## Understanding Interfaces

**An interface is a contract:**
- Defines function signatures (no implementation)
- All functions are external
- Cannot have state variables
- Cannot have constructors
- Acts as a blueprint/contract

**Why use interfaces?**
1. **Standardization** - All vaults must implement these functions
2. **Polymorphism** - Treat different vaults the same way
3. **Loose coupling** - Manager doesn't need to know implementation details
4. **Upgradability** - Swap implementations without changing interface

**Real-world examples:**
- **IERC20** - Standard token interface
- **IERC721** - Standard NFT interface
- **IUniswapV2Router** - DEX interface

---

## Understanding Abstract Contracts

**An abstract contract:**
- Can have implemented functions
- Can have unimplemented functions
- Cannot be deployed directly
- Must be inherited and completed
- Can have state variables and constructors

**Difference from interfaces:**

| Feature | Interface | Abstract Contract |
|---------|-----------|------------------|
| **Implementation** | None | Partial |
| **State variables** | No | Yes |
| **Constructor** | No | Yes |
| **Inheritance** | Multiple | Single/Multiple |
| **Use case** | Define API | Share common code |

**Why use abstract contracts?**
1. **Code reuse** - Share common logic across implementations
2. **Enforce structure** - Require children to implement specific functions
3. **Template pattern** - Provide base functionality, customize specifics

---

## The Vault System Architecture

```
IDepositBox (Interface)
    ↓
BaseDepositBox (Abstract)
    ↓
┌───────────┬──────────────┬──────────────────┐
│           │              │                  │
Basic    Premium    TimeLocked    (Future types)
```

**Benefits:**
- VaultManager works with IDepositBox interface
- BaseDepositBox provides common functionality
- Specific vaults add unique features
- Easy to add new vault types

---

## Building the System

### Step 1: Define the Interface

```solidity
interface IDepositBox {
    function getOwner() external view returns (address);
    function transferOwnership(address newOwner) external;
    function storeSecret(string calldata secret) external;
    function getSecret() external view returns (string memory);
    function getBoxType() external pure returns (string memory);
    function getDepositTime() external view returns (uint256);
}
```

**Key points:**
- All functions are `external`
- No function bodies (just signatures)
- Defines the contract that ALL vaults must follow

### Step 2: Create the Abstract Base

```solidity
abstract contract BaseDepositBox is IDepositBox {
    address private owner;
    string private secret;
    uint256 private depositTime;

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor() {
        owner = msg.sender;
        depositTime = block.timestamp;
    }

    // Implemented functions
    function storeSecret(string calldata _secret) external virtual override onlyOwner {
        secret = _secret;
    }

    function getSecret() public view virtual override onlyOwner returns (string memory) {
        return secret;
    }

    function getOwner() public view override returns (address) {
        return owner;
    }

    function transferOwnership(address newOwner) external override onlyOwner {
        require(newOwner != address(0), "Invalid address");
        owner = newOwner;
    }

    function getDepositTime() external view virtual override returns (uint256) {
        return depositTime;
    }

    // Abstract function - children MUST implement
    function getBoxType() external pure virtual override returns (string memory);
}
```

**Key points:**
- Implements most interface functions
- Provides common functionality (owner, secret storage)
- `getBoxType()` is abstract - children must implement
- Uses `virtual` to allow overriding

### Step 3: Create Specific Implementations

**BasicDepositBox:**
```solidity
contract BasicDepositBox is BaseDepositBox {
    function getBoxType() external pure override returns (string memory) {
        return "Basic";
    }
}
```

**PremiumDepositBox:**
```solidity
contract PremiumDepositBox is BaseDepositBox {
    mapping(string => string) public metadata;
    
    function setMetadata(string memory key, string memory value) external onlyOwner {
        metadata[key] = value;
    }
    
    function getBoxType() external pure override returns (string memory) {
        return "Premium";
    }
}
```

**TimeLockedDepositBox:**
```solidity
contract TimeLockedDepositBox is BaseDepositBox {
    uint256 public unlockTime;
    
    constructor(uint256 _lockDuration) {
        unlockTime = block.timestamp + _lockDuration;
    }
    
    modifier timeUnlocked() {
        require(block.timestamp >= unlockTime, "Still locked");
        _;
    }
    
    function getSecret() public view override timeUnlocked returns (string memory) {
        return super.getSecret();
    }
    
    function getBoxType() external pure override returns (string memory) {
        return "TimeLocked";
    }
}
```

### Step 4: VaultManager

```solidity
contract VaultManager {
    mapping(address => IDepositBox[]) public userVaults;
    
    event VaultCreated(address indexed user, address vault, string vaultType);
    
    function createBasicVault() external returns (address) {
        BasicDepositBox vault = new BasicDepositBox();
        userVaults[msg.sender].push(IDepositBox(address(vault)));
        emit VaultCreated(msg.sender, address(vault), "Basic");
        return address(vault);
    }
    
    function createPremiumVault() external returns (address) {
        PremiumDepositBox vault = new PremiumDepositBox();
        userVaults[msg.sender].push(IDepositBox(address(vault)));
        emit VaultCreated(msg.sender, address(vault), "Premium");
        return address(vault);
    }
    
    function createTimeLockedVault(uint256 lockDuration) external returns (address) {
        TimeLockedDepositBox vault = new TimeLockedDepositBox(lockDuration);
        userVaults[msg.sender].push(IDepositBox(address(vault)));
        emit VaultCreated(msg.sender, address(vault), "TimeLocked");
        return address(vault);
    }
    
    function getUserVaults(address user) external view returns (IDepositBox[] memory) {
        return userVaults[user];
    }
    
    function getVaultInfo(address vaultAddress) external view returns (
        string memory vaultType,
        address owner,
        uint256 depositTime
    ) {
        IDepositBox vault = IDepositBox(vaultAddress);
        return (
            vault.getBoxType(),
            vault.getOwner(),
            vault.getDepositTime()
        );
    }
}
```

**Key points:**
- Works with `IDepositBox` interface
- Doesn't need to know implementation details
- Can add new vault types easily
- Type-safe with interface

---

## Key Concepts You've Learned

**1. Interfaces** - Defining contracts without implementation

**2. Abstract contracts** - Partial implementation with shared code

**3. Polymorphism** - Treating different types through common interface

**4. Virtual/Override** - Allowing and implementing overrides

**5. Modular design** - Building extensible systems

**6. Factory pattern** - Manager creates and tracks instances

---

## Why This Matters

Interfaces and abstract contracts enable:
- **ERC standards** - IERC20, IERC721, IERC1155
- **Uniswap** - IUniswapV2Pair, IUniswapV2Router
- **Aave** - ILendingPool, ILendingPoolAddressesProvider
- **Compound** - ICToken, IComptroller

Every major protocol uses these patterns for modularity and upgradability.

## Challenge Yourself

Extend this vault system:
1. Add a MultiSigDepositBox requiring multiple approvals
2. Create a RecurringDepositBox with scheduled deposits
3. Build an NFTDepositBox for storing NFTs
4. Implement a SocialRecoveryBox with guardian system
5. Add a DAO-controlled vault type

You've mastered interfaces and abstract contracts. This is professional-level architecture!
