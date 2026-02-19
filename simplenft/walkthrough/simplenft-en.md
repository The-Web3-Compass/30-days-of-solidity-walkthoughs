# SimpleNFT Contract: Building the ERC-721 Standard from Scratch

Welcome to the **$69 billion NFT revolution**! Today, you're building a Non-Fungible Token (NFT) from scratch - the same technology that powers Bored Apes, CryptoPunks, and every digital collectible on Ethereum.

## Why NFTs Changed Everything

**Before NFTs (pre-2017):**
- Digital art could be copied infinitely
- No way to prove ownership
- Artists couldn't monetize digital work
- No secondary market royalties

**After ERC-721 (2017+):**
- Provable digital ownership
- Unique, non-duplicable tokens
- Artists earn royalties on resales
- $69B+ in trading volume

**Famous NFT sales:**
- Beeple's "Everydays": $69M
- CryptoPunk #5822: $23.7M
- Bored Ape #8817: $3.4M

## What Makes NFTs Special

**Fungible (ERC-20):**
- 1 USDC = 1 USDC (interchangeable)
- Like dollar bills
- Divisible

**Non-Fungible (ERC-721):**
- Each token is unique
- Like real estate deeds
- Indivisible
- Has metadata (image, properties)

## The ERC-721 Standard

**What is ERC-721?** A standard interface for NFTs, ensuring compatibility across wallets, marketplaces, and applications.

**Key features:**
- Unique token IDs
- Ownership tracking
- Transfer functionality
- Approval mechanism (for marketplaces)
- Metadata URIs (links to images/data)

**Why standards matter:** OpenSea, Blur, and all marketplaces work with ANY ERC-721 token automatically.

### Full Contract: SimpleNFT.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IERC721 {
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    function balanceOf(address owner) external view returns (uint256);
    function ownerOf(uint256 tokenId) external view returns (address);
    function approve(address to, uint256 tokenId) external;
    function getApproved(uint256 tokenId) external view returns (address);
    function setApprovalForAll(address operator, bool approved) external;
    function isApprovedForAll(address owner, address operator) external view returns (bool);
    function transferFrom(address from, address to, uint256 tokenId) external;
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes calldata data) external;
}

interface IERC721Receiver {
    function onERC721Received(address operator, address from, uint256 tokenId, bytes calldata data) external returns (bytes4);
}

contract SimpleNFT is IERC721 {
    string public name;
    string public symbol;
    uint256 private _tokenIdCounter = 1;
    mapping(uint256 => address) private _owners;
    mapping(address => uint256) private _balances;
    mapping(uint256 => address) private _tokenApprovals;
    mapping(address => mapping(address => bool)) private _operatorApprovals;
    mapping(uint256 => string) private _tokenURIs;

    constructor(string memory name_, string memory symbol_) {
        name = name_;
        symbol = symbol_;
    }

    function balanceOf(address owner) public view override returns (uint256) {
        require(owner != address(0), "Zero address");
        return _balances[owner];
    }

    function ownerOf(uint256 tokenId) public view override returns (address) {
        address owner = _owners[tokenId];
        require(owner != address(0), "Token doesn't exist");
        return owner;
    }

    function approve(address to, uint256 tokenId) public override {
        address owner = ownerOf(tokenId);
        require(to != owner, "Already owner");
        require(msg.sender == owner || isApprovedForAll(owner, msg.sender), "Not authorized");
        _tokenApprovals[tokenId] = to;
        emit Approval(owner, to, tokenId);
    }

    function getApproved(uint256 tokenId) public view override returns (address) {
        require(_owners[tokenId] != address(0), "Token doesn't exist");
        return _tokenApprovals[tokenId];
    }

    function setApprovalForAll(address operator, bool approved) public override {
        require(operator != msg.sender, "Self approval");
        _operatorApprovals[msg.sender][operator] = approved;
        emit ApprovalForAll(msg.sender, operator, approved);
    }

    function isApprovedForAll(address owner, address operator) public view override returns (bool) {
        return _operatorApprovals[owner][operator];
    }

    function transferFrom(address from, address to, uint256 tokenId) public override {
        require(_isApprovedOrOwner(msg.sender, tokenId), "Not authorized");
        _transfer(from, to, tokenId);
    }

    function safeTransferFrom(address from, address to, uint256 tokenId) public override {
        safeTransferFrom(from, to, tokenId, "");
    }

    function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory data) public override {
        require(_isApprovedOrOwner(msg.sender, tokenId), "Not authorized");
        _safeTransfer(from, to, tokenId, data);
    }

    function mint(address to, string memory uri) public {
        uint256 tokenId = _tokenIdCounter;
        _tokenIdCounter++;
        _owners[tokenId] = to;
        _balances[to] += 1;
        _tokenURIs[tokenId] = uri;
        emit Transfer(address(0), to, tokenId);
    }

    function tokenURI(uint256 tokenId) public view returns (string memory) {
        require(_owners[tokenId] != address(0), "Token doesn't exist");
        return _tokenURIs[tokenId];
    }

    function _transfer(address from, address to, uint256 tokenId) internal virtual {
        require(ownerOf(tokenId) == from, "Not owner");
        require(to != address(0), "Zero address");
        _balances[from] -= 1;
        _balances[to] += 1;
        _owners[tokenId] = to;
        delete _tokenApprovals[tokenId];
        emit Transfer(from, to, tokenId);
    }

    function _safeTransfer(address from, address to, uint256 tokenId, bytes memory data) internal virtual {
        _transfer(from, to, tokenId);
        require(_checkOnERC721Received(from, to, tokenId, data), "Not ERC721Receiver");
    }

    function _isApprovedOrOwner(address spender, uint256 tokenId) internal view returns (bool) {
        address owner = ownerOf(tokenId);
        return (spender == owner || getApproved(tokenId) == spender || isApprovedForAll(owner, spender));
    }

    function _checkOnERC721Received(address from, address to, uint256 tokenId, bytes memory data) private returns (bool) {
        if (to.code.length > 0) {
            try IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, data) returns (bytes4 retval) {
                return retval == IERC721Receiver.onERC721Received.selector;
            } catch {
                return false;
            }
        }
        return true;
    }
}
```

---

### Step-by-Step Breakdown

#### 1. The Data Structures
An NFT is really just a number (Token ID) assigned to an owner.
*   `_owners`: Using a mapping, we link a `tokenId` (uint256) to an `address`.
*   `_balances`: How many NFTs does this person own?
*   `_tokenURIs`: Where is the image stored? (Usually an IPFS link).
```solidity
mapping(uint256 => address) private _owners;
mapping(address => uint256) private _balances;
mapping(uint256 => string) private _tokenURIs;
```

#### 2. The Golden Rule: "Not your keys, not your crypto"
We need to ensure only the owner (or someone they approved) can move the NFT.
```solidity
function transferFrom(address from, address to, uint256 tokenId) public override {
    require(_isApprovedOrOwner(msg.sender, tokenId), "Not authorized");
    _transfer(from, to, tokenId);
}
```

#### 3. Approvals: The Marketplace Power
If you want to sell your NFT on OpenSea, you don't send it to them. You **approve** them to take it when a buyer pays.
```solidity
function approve(address to, uint256 tokenId) public override {
    // Check if msg.sender is the owner...
    _tokenApprovals[tokenId] = to;
}
```

#### 4. Minting: Creating Art
This is how new NFTs are born.
```solidity
function mint(address to, string memory uri) public {
    uint256 tokenId = _tokenIdCounter; // Get the next ID (e.g., #1)
    _tokenIdCounter++;                 // Increment for next time
    _owners[tokenId] = to;             // Assign ownership
    _balances[to] += 1;                // Update count
    _tokenURIs[tokenId] = uri;         // Set the metadata link
    emit Transfer(address(0), to, tokenId); // Announce it to the world!
}
```

Congratulations! You just wrote the logic that powers billions of dollars in digital asset trading.

---

## Understanding NFT Metadata

### The Token URI

```solidity
mapping(uint256 => string) private _tokenURIs;

function tokenURI(uint256 tokenId) public view returns (string memory) {
    return _tokenURIs[tokenId];
}
```

**What is a token URI?** A link to JSON metadata describing the NFT.

**Example URI:** `ipfs://QmX.../metadata.json`

**Example metadata:**
```json
{
  "name": "Cool NFT #1",
  "description": "A very cool NFT",
  "image": "ipfs://QmY.../image.png",
  "attributes": [
    {"trait_type": "Background", "value": "Blue"},
    {"trait_type": "Rarity", "value": "Legendary"}
  ]
}
```

**Why IPFS?** Decentralized storage - images can't be taken down or changed.

---

## The Approval System Explained

### Why Approvals Exist

**Problem:** You want to sell your NFT on OpenSea, but you don't want to send it to them.

**Solution:** Approve OpenSea to transfer it when someone buys.

### Single Token Approval

```solidity
function approve(address to, uint256 tokenId) public override {
    _tokenApprovals[tokenId] = to;
}
```

**Usage:**
```solidity
// Approve OpenSea for token #5
nft.approve(openSeaAddress, 5);
```

### Operator Approval (All Tokens)

```solidity
function setApprovalForAll(address operator, bool approved) public override {
    _operatorApprovals[msg.sender][operator] = approved;
}
```

**Usage:**
```solidity
// Approve OpenSea for ALL your tokens
nft.setApprovalForAll(openSeaAddress, true);
```

**Why this exists:** Convenience - approve once, list many NFTs.

---

## SafeTransferFrom vs TransferFrom

### transferFrom()

```solidity
function transferFrom(address from, address to, uint256 tokenId) public override {
    require(_isApprovedOrOwner(msg.sender, tokenId));
    _transfer(from, to, tokenId);
}
```

**Problem:** If `to` is a contract that doesn't handle NFTs, the NFT gets stuck forever!

### safeTransferFrom()

```solidity
function safeTransferFrom(address from, address to, uint256 tokenId) public override {
    require(_isApprovedOrOwner(msg.sender, tokenId));
    _safeTransfer(from, to, tokenId, "");
}
```

**Protection:** Checks if recipient is a contract, and if so, verifies it can handle NFTs.

```solidity
function _checkOnERC721Received(...) private returns (bool) {
    if (to.code.length > 0) {
        // It's a contract - check if it implements onERC721Received
        try IERC721Receiver(to).onERC721Received(...) returns (bytes4 retval) {
            return retval == IERC721Receiver.onERC721Received.selector;
        } catch {
            return false;
        }
    }
    return true; // Not a contract, safe to send
}
```

**Always use safeTransferFrom** to prevent accidental NFT loss!

---

## Key Concepts You've Learned

**1. ERC-721 standard** - The NFT interface specification

**2. Unique token IDs** - Each NFT is distinct

**3. Ownership tracking** - Mapping tokens to owners

**4. Approval system** - Enabling marketplace interactions

**5. Metadata URIs** - Linking to off-chain data

**6. Safe transfers** - Preventing NFT loss

---

## Real-World NFT Implementations

### Bored Ape Yacht Club

**Features:**
- 10,000 unique apes
- Provenance tracking
- Commercial rights to holders
- Exclusive club membership

### CryptoPunks

**Unique:** Pre-ERC-721 (custom contract)
- 10,000 24x24 pixel art characters
- Wrapped versions for ERC-721 compatibility

### ENS (Ethereum Name Service)

**NFTs as utility:**
- Domain names as NFTs
- Transferable, tradeable
- Functional (resolve to addresses)

---

## Advanced NFT Features

### 1. Enumerable Extension

```solidity
// Track all tokens owned by an address
mapping(address => uint256[]) private _ownedTokens;

function tokenOfOwnerByIndex(address owner, uint256 index) public view returns (uint256) {
    return _ownedTokens[owner][index];
}
```

### 2. Royalties (EIP-2981)

```solidity
function royaltyInfo(uint256 tokenId, uint256 salePrice) 
    external view returns (address receiver, uint256 royaltyAmount) 
{
    return (creator, salePrice * 5 / 100); // 5% royalty
}
```

### 3. Soulbound Tokens

```solidity
// Non-transferable NFTs
function _transfer(address from, address to, uint256 tokenId) internal override {
    revert("Token is soulbound");
}
```

---

## Why This Matters

NFTs enable:
- **Digital ownership** - Provable scarcity
- **Creator economy** - Artists earn from resales
- **Gaming** - True ownership of in-game items
- **Identity** - On-chain credentials and reputation
- **Real estate** - Tokenized property deeds

The ERC-721 standard created a $69B+ industry and fundamentally changed how we think about digital ownership.

## Challenge Yourself

Extend this NFT contract:
1. Add enumerable functionality (track all tokens)
2. Implement EIP-2981 royalties
3. Create a whitelist minting system
4. Add reveal mechanics (hidden metadata)
5. Build a breeding/evolution system

You've mastered NFT development. This is the foundation of the digital collectibles revolution!
