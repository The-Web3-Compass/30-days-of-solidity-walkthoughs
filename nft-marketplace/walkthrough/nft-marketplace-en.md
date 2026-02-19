# NFT Marketplace: Building Your Own OpenSea

Welcome to one of the most exciting projects in Web3: building an **NFT marketplace**. Today, you're creating the core technology that powers OpenSea ($30B+ in volume), Blur, LooksRare, and every other NFT platform.

## The NFT Marketplace Revolution

**The market:**
- OpenSea: $30B+ total volume
- Blur: $10B+ total volume
- Magic Eden: $3B+ total volume
- Thousands of collections traded daily

**What makes marketplaces valuable:**
- They don't hold the NFTs (just facilitate trades)
- They take a small fee (2-5%) on every sale
- They're composable (other apps can integrate)
- They enable price discovery and liquidity

## What We're Building

A complete NFT marketplace with:
- **Listing system** - Sellers list NFTs with prices
- **Buying mechanism** - Buyers purchase with ETH
- **Approval pattern** - NFTs stay in seller's wallet until sold
- **Fee structure** - Platform takes a percentage
- **Cancellation** - Sellers can delist anytime
- **Multi-collection support** - Works with any ERC721

This is production-level marketplace logic!

### Full NFT Marketplace Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract NFTMarketplace is ReentrancyGuard {
    address public owner;
    uint256 public marketplaceFeePercent = 250; // 2.5% fee (basis points)
    address public feeRecipient;

    struct Listing {
        address seller;
        address nftAddress;
        uint256 tokenId;
        uint256 price;
        bool isListed;
    }

    // NFT Address -> Token ID -> Listing
    mapping(address => mapping(uint256 => Listing)) public listings;
    
    // Revenue collected by the marketplace
    uint256 public feesCollected;

    event ItemListed(address indexed seller, address indexed nftAddress, uint256 indexed tokenId, uint256 price);
    event ItemCanceled(address indexed seller, address indexed nftAddress, uint256 indexed tokenId);
    event ItemBought(address indexed buyer, address indexed nftAddress, uint256 indexed tokenId, uint256 price);

    constructor(address _feeRecipient) {
        owner = msg.sender;
        feeRecipient = _feeRecipient;
    }

    /////////////////////
    // MAIN FUNCTIONS //
    ////////////////////

    /*
     * @notice Method for listing your NFT on the marketplace
     * @param nftAddress: Address of the NFT
     * @param tokenId: The Token ID of the NFT
     * @param price: sale price of the listed NFT
     */
    function listItem(
        address nftAddress,
        uint256 tokenId,
        uint256 price
    ) external nonReentrant {
        require(price > 0, "Price must be greater than 0");
        
        // 1. Check if the marketplace is approved to move the NFT
        IERC721 nft = IERC721(nftAddress);
        require(nft.getApproved(tokenId) == address(this) || nft.isApprovedForAll(msg.sender, address(this)), "Not approved for marketplace");
        
        // 2. Check ownership
        require(nft.ownerOf(tokenId) == msg.sender, "Not the owner");

        // 3. Create the listing
        listings[nftAddress][tokenId] = Listing(msg.sender, nftAddress, tokenId, price, true);
        
        emit ItemListed(msg.sender, nftAddress, tokenId, price);
    }

    /*
     * @notice Method for buying a listing
     * @param nftAddress: Address of the NFT
     * @param tokenId: The Token ID of the NFT
     */
    function buyItem(address nftAddress, uint256 tokenId) external payable nonReentrant {
        Listing memory listedItem = listings[nftAddress][tokenId];
        require(listedItem.isListed, "Item not listed");
        require(msg.value == listedItem.price, "Price not met");

        // 1. Calculate Fees
        // Fee for the marketplace
        uint256 feeAmount = (msg.value * marketplaceFeePercent) / 10000;
        // Amount for the seller
        uint256 sellerAmount = msg.value - feeAmount;

        // 2. Update State (Delete listing BEFORE transfer to prevent reentrancy)
        delete listings[nftAddress][tokenId];
        feesCollected += feeAmount;

        // 3. Transfer Money
        // Pay the seller
        (bool success, ) = payable(listedItem.seller).call{value: sellerAmount}("");
        require(success, "Transfer to seller failed");

        // 4. Transfer NFT
        IERC721(nftAddress).safeTransferFrom(listedItem.seller, msg.sender, tokenId);

        emit ItemBought(msg.sender, nftAddress, tokenId, listedItem.price);
    }

    function cancelListing(address nftAddress, uint256 tokenId) external nonReentrant {
        Listing memory listedItem = listings[nftAddress][tokenId];
        require(listedItem.seller == msg.sender, "Not the seller");
        require(listedItem.isListed, "Not listed");

        delete listings[nftAddress][tokenId];
        emit ItemCanceled(msg.sender, nftAddress, tokenId);
    }

    function withdrawFees() external {
        require(msg.sender == owner, "Only owner");
        (bool success, ) = payable(feeRecipient).call{value: address(this).balance}("");
        require(success, "Transfer failed");
    }
}
```

---

### Understanding the Logic

#### 1. The Listing Struct
We need to store data for every item listed.
```solidity
struct Listing {
    address seller;      // Who is selling it?
    uint256 price;       // How much?
    bool isListed;       // Is it active?
}
```
We use a **nested mapping** to access this: `listings[ContractAddress][TokenID]`. This way, we can support *any* NFT collection on the blockchain, not just one!

#### 2. The Approval Pattern
Notice we don't transfer the NFT to the marketplace when listing. We keep it in the user's wallet.
However, the user MUST call `approve()` on the NFT contract first.
```solidity
// In listNFT check:
require(nft.getApproved(tokenId) == address(this), "Not approved");
```
This gives our marketplace permission to pull the NFT *only* when a buyer pays.

#### 3. Checks-Effects-Interactions Pattern

In `buyItem`, we follow the most important security pattern in Solidity:

```solidity
function buyItem(address nftAddress, uint256 tokenId) external payable nonReentrant {
    // 1. CHECKS - Validate everything first
    Listing memory listedItem = listings[nftAddress][tokenId];
    require(listedItem.isListed, "Item not listed");
    require(msg.value == listedItem.price, "Price not met");
    
    // 2. EFFECTS - Update state BEFORE external calls
    delete listings[nftAddress][tokenId];
    feesCollected += feeAmount;
    
    // 3. INTERACTIONS - External calls LAST
    (bool success, ) = payable(listedItem.seller).call{value: sellerAmount}("");
    require(success, "Transfer to seller failed");
    IERC721(nftAddress).safeTransferFrom(listedItem.seller, msg.sender, tokenId);
}
```

**Why this order matters:**
- If we transferred ETH BEFORE updating state, a malicious seller could re-enter and drain funds
- If we transferred the NFT BEFORE deleting the listing, someone could buy it twice
- This pattern prevents reentrancy attacks

---

## Understanding the Approval Pattern

### Why NFTs Stay in Your Wallet

**Traditional marketplace (centralized):**
1. Send NFT to marketplace
2. Marketplace holds it
3. Marketplace sends to buyer

**Problems:**
- Marketplace could get hacked
- Marketplace could steal NFTs
- You lose custody

**Decentralized marketplace (our approach):**
1. Keep NFT in your wallet
2. Approve marketplace to move it
3. Marketplace only moves it when sold

**Benefits:**
- You maintain custody until sale
- Marketplace can't steal
- Can list on multiple marketplaces
- Can cancel anytime

### The Approval Flow

**Step 1: Approve the marketplace**
```solidity
// User calls this on the NFT contract
nft.approve(marketplaceAddress, tokenId);
// "Marketplace, you can move this specific NFT"
```

**Or approve all:**
```solidity
nft.setApprovalForAll(marketplaceAddress, true);
// "Marketplace, you can move ANY of my NFTs"
```

**Step 2: List on marketplace**
```solidity
marketplace.listItem(nftAddress, tokenId, price);
```

**Step 3: Marketplace checks approval**
```solidity
require(
    nft.getApproved(tokenId) == address(this) || 
    nft.isApprovedForAll(msg.sender, address(this)),
    "Not approved"
);
```

**Step 4: When sold, marketplace moves NFT**
```solidity
IERC721(nftAddress).safeTransferFrom(seller, buyer, tokenId);
```

---

## Fee Structures Explained

### Platform Fees

```solidity
uint256 public marketplaceFeePercent = 250; // 2.5% in basis points

function buyItem() external payable {
    uint256 feeAmount = (msg.value * marketplaceFeePercent) / 10000;
    uint256 sellerAmount = msg.value - feeAmount;
    
    // Platform keeps feeAmount
    // Seller gets sellerAmount
}
```

**Real-world marketplace fees:**
- **OpenSea:** 2.5%
- **Blur:** 0% (subsidized by token)
- **LooksRare:** 2%
- **X2Y2:** 0.5%

**Why fees matter:**
- Lower fees = more volume
- But need revenue to sustain
- Competitive landscape

### Creator Royalties

```solidity
function buyItem() external payable {
    // Calculate fees
    uint256 platformFee = (msg.value * 250) / 10000; // 2.5%
    uint256 royaltyFee = (msg.value * royaltyPercent) / 10000; // e.g., 5%
    uint256 sellerAmount = msg.value - platformFee - royaltyFee;
    
    // Pay everyone
    payable(creator).transfer(royaltyFee);
    payable(seller).transfer(sellerAmount);
}
```

**The royalty debate:**
- **Pro:** Creators earn on secondary sales
- **Con:** Can't be enforced on-chain
- **Reality:** Many marketplaces made royalties optional
- **Impact:** Creators lost significant revenue

---

## Advanced Marketplace Features

### 1. Offers/Bidding

```solidity
struct Offer {
    address bidder;
    uint256 amount;
    uint256 expiration;
}

mapping(address => mapping(uint256 => Offer[])) public offers;

function makeOffer(address nftAddress, uint256 tokenId) external payable {
    offers[nftAddress][tokenId].push(Offer({
        bidder: msg.sender,
        amount: msg.value,
        expiration: block.timestamp + 7 days
    }));
}

function acceptOffer(address nftAddress, uint256 tokenId, uint256 offerIndex) external {
    // Seller accepts a specific offer
}
```

### 2. Auctions

```solidity
struct Auction {
    address seller;
    uint256 startPrice;
    uint256 highestBid;
    address highestBidder;
    uint256 endTime;
}

function createAuction(address nftAddress, uint256 tokenId, uint256 startPrice, uint256 duration) external {
    // Create timed auction
}

function placeBid(address nftAddress, uint256 tokenId) external payable {
    // Place bid, refund previous bidder
}
```

### 3. Bundle Sales

```solidity
struct Bundle {
    address[] nftAddresses;
    uint256[] tokenIds;
    uint256 price;
}

function listBundle(address[] memory nfts, uint256[] memory ids, uint256 price) external {
    // List multiple NFTs as one package
}
```

---

## Security Considerations

### 1. Reentrancy Protection

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract NFTMarketplace is ReentrancyGuard {
    function buyItem() external payable nonReentrant {
        // Protected from reentrancy
    }
}
```

### 2. Price Manipulation

**The problem:** Seller could front-run a buy and change the price.

**The solution:** Buyers specify exact price they're willing to pay:
```solidity
require(msg.value == listedItem.price, "Price not met");
```

### 3. Stale Listings

**The problem:** NFT gets transferred, but listing remains.

**The solution:** Check ownership before sale:
```solidity
require(nft.ownerOf(tokenId) == listedItem.seller, "Seller no longer owns NFT");
```

### 4. Approval Revocation

**The problem:** Seller revokes approval but listing remains.

**The solution:** Check approval before transfer:
```solidity
require(
    nft.getApproved(tokenId) == address(this),
    "Marketplace not approved"
);
```

---

## Gas Optimization Tips

### 1. Use Memory for Structs

```solidity
// GOOD - Load once into memory
Listing memory item = listings[nftAddress][tokenId];
uint256 price = item.price;
address seller = item.seller;

// BAD - Multiple storage reads
uint256 price = listings[nftAddress][tokenId].price;
address seller = listings[nftAddress][tokenId].seller;
```

### 2. Delete Listings

```solidity
delete listings[nftAddress][tokenId];
// Gets gas refund for clearing storage
```

### 3. Batch Operations

```solidity
function listMultiple(
    address[] calldata nfts,
    uint256[] calldata ids,
    uint256[] calldata prices
) external {
    for (uint i = 0; i < nfts.length; i++) {
        listItem(nfts[i], ids[i], prices[i]);
    }
}
```

---

## Real-World Marketplace Comparison

### OpenSea
- **Fee:** 2.5%
- **Features:** Offers, auctions, bundles, collections
- **Unique:** Lazy minting, gasless listings

### Blur
- **Fee:** 0% (token incentives)
- **Features:** Aggregated liquidity, advanced trading
- **Unique:** Trader rewards, portfolio management

### LooksRare
- **Fee:** 2%
- **Features:** Staking rewards, private sales
- **Unique:** LOOKS token rewards for trading

---

## Key Concepts You've Learned

**1. Approval pattern** - NFTs stay in seller's wallet

**2. Nested mappings** - Support any NFT collection

**3. Fee structures** - Platform and creator royalties

**4. Checks-Effects-Interactions** - Security pattern

**5. ReentrancyGuard** - Protection from attacks

**6. Multi-collection support** - Universal marketplace

---

## Why This Matters

You just built the core of a **multi-billion dollar industry**:
- NFT marketplaces generated $25B+ in fees (2021-2023)
- OpenSea alone: $30B+ in volume
- Blur: $10B+ in volume
- This is real-world, production technology

## Challenge Yourself

Extend your marketplace:
1. Add offer/bidding system
2. Implement English auctions
3. Add bundle sales (multiple NFTs)
4. Create collection-wide offers
5. Build aggregated liquidity (buy from multiple marketplaces)

You've mastered NFT marketplace development. This is cutting-edge Web3 technology!

---

You have now built a platform where digital assets can trade hands securely and trustlessly. 🤝
