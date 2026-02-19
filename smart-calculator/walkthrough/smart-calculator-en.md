# Smart Calculator: Mastering Inter-Contract Communication

Welcome to **inter-contract communication** - one of the most powerful patterns in Solidity. Today, you're learning how contracts can call each other, enabling modular, upgradeable, and composable systems.

## Why Contracts Need to Talk

**The monolithic problem:**
```solidity
contract MassiveCalculator {
    // 5000+ lines
    // Basic math + scientific functions + statistics + graphing
    // Can't upgrade without redeploying everything
    // Hits size limits
}
```

**The modular solution:**
```solidity
contract Calculator {
    // Basic math (100 lines)
    // Calls ScientificCalculator when needed
}

contract ScientificCalculator {
    // Advanced math (200 lines)
    // Reusable by multiple contracts
}
```

**Benefits:**
- Each contract stays under size limits
- Upgrade individual components
- Reuse logic across projects
- Better testing and auditing

**Real-world examples:**
- **Uniswap:** Router calls Pair contracts
- **Aave:** LendingPool calls multiple modules
- **Compound:** Comptroller orchestrates CToken contracts

### File 1: ScientificCalculator.sol
This contract is pure logic. It doesn't store data, it just computes.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ScientificCalculator {
    function power(uint256 base, uint256 exponent) public pure returns (uint256) {
        if (exponent == 0) return 1;
        else return (base ** exponent);
    }

    function squareRoot(uint256 number) public pure returns (uint256) {
        require(number >= 0, "Cannot calculate square root of negative number");
        if (number == 0) return 0;

        uint256 result = number / 2;
        for (uint256 i = 0; i < 10; i++) {
            result = (result + number / result) / 2;
        }
        return result;
    }
}
```

### File 2: Calculator.sol
This is the main contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./ScientificCalculator.sol";

contract Calculator {
    address public owner;
    address public scientificCalculatorAddress;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can perform this action");
        _;
    }

    function setScientificCalculator(address _address) public onlyOwner {
        scientificCalculatorAddress = _address;
    }

    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }

    function subtract(uint256 a, uint256 b) public pure returns (uint256) {
        return a - b;
    }

    function multiply(uint256 a, uint256 b) public pure returns (uint256) {
        return a * b;
    }

    function divide(uint256 a, uint256 b) public pure returns (uint256) {
        require(b != 0, "Cannot divide by zero");
        return a / b;
    }

    // INTERFACE CALL
    function calculatePower(uint256 base, uint256 exponent) public view returns (uint256) {
        ScientificCalculator scientificCalc = ScientificCalculator(scientificCalculatorAddress);
        return scientificCalc.power(base, exponent);
    }

    // LOW-LEVEL CALL
    function calculateSquareRoot(uint256 number) public returns (uint256) {
        bytes memory data = abi.encodeWithSignature("squareRoot(uint256)", number);
        (bool success, bytes memory returnData) = scientificCalculatorAddress.call(data);
        require(success, "External call failed");
        return abi.decode(returnData, (uint256));
    }
}
```

---

### Step-by-Step Breakdown

#### 1. Connecting the Contracts
In `Calculator.sol`, we have a variable to store the address of our helper contract.
```solidity
address public scientificCalculatorAddress;

function setScientificCalculator(address _address) public onlyOwner {
    scientificCalculatorAddress = _address;
}
```
We deploy both contracts, get the address of `ScientificCalculator`, and save it here.

#### 2. Method A: The Interface Way (Easy)
Ideally, we import the contract definition and treat it like an object.
```solidity
function calculatePower(...) {
    // Cast the address to the Contract Type
    ScientificCalculator scientificCalc = ScientificCalculator(scientificCalculatorAddress); 
    // Call the function normally
    return scientificCalc.power(base, exponent);
}
```

#### 3. Method B: The Low-Level Way (Harder but Flexible)
Sometimes you don't have the source code of the other contract. You can still call it if you know the function signature!
```solidity
function calculateSquareRoot(uint256 number) public returns (uint256) {
    // Manually encode the function call
    bytes memory data = abi.encodeWithSignature("squareRoot(uint256)", number);
    
    // Send the "call"
    (bool success, bytes memory returnData) = scientificCalculatorAddress.call(data);
    
    require(success, "External call failed");
    // Decode the answer
    return abi.decode(returnData, (uint256));
}
```
This pattern is used heavily in proxies and upgradeable contracts!

---

## Key Concepts You've Learned

**1. Inter-contract communication** - Contracts calling other contracts

**2. Interface method** - Type-safe calls with imported contracts

**3. Low-level calls** - Dynamic calls without source code

**4. Modular architecture** - Separating concerns across contracts

**5. Upgradeability** - Swapping implementations

**6. abi.encodeWithSignature** - Manual function encoding

---

## Real-World Communication Patterns

### Uniswap Router → Pair

```solidity
contract UniswapRouter {
    function swapExactTokensForTokens(...) external {
        // Router calls Pair contract
        IUniswapV2Pair pair = IUniswapV2Pair(pairAddress);
        pair.swap(amount0Out, amount1Out, to, data);
    }
}
```

### Aave LendingPool → AToken

```solidity
contract LendingPool {
    function deposit(address asset, uint256 amount) external {
        // LendingPool calls AToken
        IAToken(aToken).mint(msg.sender, amount);
    }
}
```

---

## Why This Matters

Inter-contract communication enables:
- **Composability** - DeFi "money legos"
- **Modularity** - Separate concerns
- **Upgradeability** - Swap implementations
- **Reusability** - Share logic across projects

This is how complex DeFi protocols are built!

## Challenge Yourself

Extend this calculator system:
1. Add more mathematical operations
2. Create a Statistics contract with mean/median functions
3. Build a GraphingCalculator that uses both contracts
4. Implement access control for calculator functions
5. Add a proxy pattern for upgradeability

You've mastered inter-contract communication. This is professional-level architecture!
