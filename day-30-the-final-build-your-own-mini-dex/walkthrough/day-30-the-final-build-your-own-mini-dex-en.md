# Day 30 – The Final Build: Your Own Mini DEX

Congratulations! You made it. 🏆

For 29 days, you’ve learned the alphabet.
Today, we write the novel.

We are building a **Mini Decentralized Exchange (DEX)**.
It will have two main contracts:
1.  **Factory**: Creates new trading pairs (e.g., ETH/DAI, WBTC/USDC).
2.  **Pair**: Holds the liquidity and handles the swaps (x * y = k).

This is a simplified version of **Uniswap V2**.

---

### Part 1: The Pair (MiniDexPair.sol)

This contract holds the tokens. It has one job: strictly follow the math.
`Amount X * Amount Y = Constant K`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MiniDexPair is ERC20, ReentrancyGuard {
    address public token0;
    address public token1;

    uint256 public reserve0;
    uint256 public reserve1;

    constructor(address _token0, address _token1) ERC20("MiniDex-LP", "MDX-LP") {
        token0 = _token0;
        token1 = _token1;
    }

    // ADD LIQUIDITY
    function addLiquidity(uint256 amount0, uint256 amount1) external returns (uint256 liquidity) {
        // Transfer tokens in
        IERC20(token0).transferFrom(msg.sender, address(this), amount0);
        IERC20(token1).transferFrom(msg.sender, address(this), amount1);

        uint256 totalSupply = totalSupply();
        
        if (totalSupply == 0) {
            // Initial liquidity provider gets 100% of the pool shares
            liquidity = sqrt(amount0 * amount1);
        } else {
            // Subsequent providers get proportional shares
            liquidity = min(
                (amount0 * totalSupply) / reserve0,
                (amount1 * totalSupply) / reserve1
            );
        }

        require(liquidity > 0, "Insufficient liquidity minted");
        _mint(msg.sender, liquidity);
        
        _update(IERC20(token0).balanceOf(address(this)), IERC20(token1).balanceOf(address(this)));
    }

    // SWAP: Trade one token for another
    function swap(uint256 amount0Out, uint256 amount1Out, address to) external nonReentrant {
        require(amount0Out > 0 || amount1Out > 0, "Insufficient output amount");
        require(amount0Out < reserve0 && amount1Out < reserve1, "Insufficient liquidity");

        if (amount0Out > 0) IERC20(token0).transfer(to, amount0Out);
        if (amount1Out > 0) IERC20(token1).transfer(to, amount1Out);

        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));

        // The Magic Formula: x * y >= k
        // We require the new K to be at least as big as the old K (simplified constant product check)
        // In reality, Uniswap subtracts fees here. We'll skip fees for simplicity.
        require(balance0 * balance1 >= reserve0 * reserve1, "K");

        _update(balance0, balance1);
    }

    // REMOVE LIQUIDITY
    function removeLiquidity(uint256 liquidity) external returns (uint256 amount0, uint256 amount1) {
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 totalSupply = totalSupply();

        amount0 = (liquidity * balance0) / totalSupply;
        amount1 = (liquidity * balance1) / totalSupply;

        require(amount0 > 0 && amount1 > 0, "Insufficient amount");

        _burn(msg.sender, liquidity);
        IERC20(token0).transfer(msg.sender, amount0);
        IERC20(token1).transfer(msg.sender, amount1);

        _update(IERC20(token0).balanceOf(address(this)), IERC20(token1).balanceOf(address(this)));
    }

    function _update(uint256 balance0, uint256 balance1) private {
        reserve0 = balance0;
        reserve1 = balance1;
    }

    // Math Helpers
    function sqrt(uint256 y) internal pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) { z = x; x = (y / x + x) / 2; }
        } else if (y != 0) { z = 1; }
    }

    function min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a < b ? a : b;
    }
}
```

---

### Part 2: The Factory (MiniDexFactory.sol)

We need a way to deploy these pairs. We can't deploy them manually every time.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./MiniDexPair.sol";

contract MiniDexFactory {
    // Mapping from TokenA -> TokenB -> Pair Address
    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;

    event PairCreated(address indexed token0, address indexed token1, address pair, uint256);

    function createPair(address tokenA, address tokenB) external returns (address pair) {
        require(tokenA != tokenB, "Identical addresses");
        require(tokenA != address(0) && tokenB != address(0), "Zero address");
        require(getPair[tokenA][tokenB] == address(0), "Pair already exists");

        // Sort tokens so token0 < token1 (avoids duplicates like A-B and B-A)
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);

        // Deploy!
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        // Note: In real production we use create2 for deterministic addresses, 
        // but 'new' keyword is fine for our tutorial.
        MiniDexPair newPair = new MiniDexPair(token0, token1); 
        pair = address(newPair);

        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // Populate reverse mapping
        allPairs.push(pair);

        emit PairCreated(token0, token1, pair, allPairs.length);
    }
}
```

---

### How it all fits together

1.  **Deployment:** You deploy ONE Factory.
2.  **Creation:** Anyone calls `factory.createPair(TokenA, TokenB)`. The factory deploys a new `MiniDexPair` contract.
3.  **Liquidity:** Users call `addLiquidity` on the Pair contract to deposit tokens. They get "LP Tokens" in return.
4.  **Trading:** Traders call `swap` on the Pair contract. They send Token A and get Token B back. The price is determined automatically by the ratio of tokens in the pool.

You have now built the financial infrastructure of the future.
If you understand this, you understand DeFi.

**Course Completed.** 🎉
