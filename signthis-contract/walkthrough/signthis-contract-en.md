# SignThis Contract: Mastering Cryptographic Signatures

Welcome to one of the most powerful patterns in Web3: **off-chain signatures**. Today, you're learning how to verify identity and permissions without storing expensive data on-chain - the technology that powers airdrops, gasless transactions, and NFT whitelists.

## The Gas Problem

**Scenario:** You're launching an exclusive NFT drop with 10,000 whitelisted addresses.

**The naive approach:**
```solidity
mapping(address => bool) public whitelist;

function addToWhitelist(address[] memory addresses) public onlyOwner {
    for (uint i = 0; i < addresses.length; i++) {
        whitelist[addresses[i]] = true;
    }
}
```

**The cost:**
- Each address: ~20,000 gas
- 10,000 addresses: 200,000,000 gas
- At 50 gwei: ~$500-1,000+
- **And you pay it all upfront!**

**The problem:**
- Expensive to deploy
- Can't easily update the list
- Storage bloat on-chain
- You pay for everyone's verification

## The Signature Solution

**Instead of storing the whitelist:**
1. **Off-chain:** Sign each address with your private key (free!)
2. **Give signature to user:** They store it (not you)
3. **On-chain:** User presents signature, contract verifies it
4. **Cost:** 0 gas for you, normal gas for user

**Benefits:**
- No upfront storage costs
- Infinitely scalable
- Easy to update (just sign new addresses)
- Users pay their own gas
- Privacy-preserving (whitelist not public)

**Real-world usage:**
- **OpenSea:** Gasless listings with signatures
- **Uniswap Permit:** Gasless token approvals
- **NFT Mints:** Whitelist verification
- **Airdrops:** Merkle trees + signatures
- **Meta-transactions:** Gasless interactions

### EventEntry.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

contract EventEntry {
    using ECDSA for bytes32;
    using MessageHashUtils for bytes32;

    address public organizer;
    mapping(address => bool) public hasEntered;

    constructor() {
        organizer = msg.sender;
    }

    /* 
     * @dev User provides a signature. We verify if the 'organizer' signed it.
     * The message signed is simply the user's address.
     */
    function enterEvent(bytes memory signature) external {
        require(!hasEntered[msg.sender], "Already entered");

        // 1. Recreate the hash that was signed
        // We hash the msg.sender because the permission slip is bound to THEIR address.
        // They can't give it to someone else.
        bytes32 messageHash = keccak256(abi.encodePacked(msg.sender));
        
        // 2. Add the "Ethereum Signed Message" prefix (standard security practice)
        bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();

        // 3. Recover the signer address from the signature
        address signer = ethSignedMessageHash.recover(signature);

        // 4. Check if it matches the organizer
        require(signer == organizer, "Invalid signature");

        // 5. Success!
        hasEntered[msg.sender] = true;
    }
}
```

---

### How to use this DApp?
1.  **Backend:** You run a script (using ethers.js or web3.js). When a user qualifies, you sign their address with your private key.
    ```javascript
    // JS Example
    const hash = ethers.utils.solidityKeccak256(["address"], [userAddress]);
    const signature = await wallet.signMessage(ethers.utils.arrayify(hash));
    ```
2.  **Frontend:** You give that `signature` to the user.
3.  **Contract:** The user calls `enterEvent(signature)`. The contract sees your specific signature and approves the transaction.

---

## Understanding Cryptographic Signatures

### What is a Digital Signature?

**In the real world:**
- You sign a document with a pen
- Your signature proves you approved it
- Hard to forge (but not impossible)

**In crypto:**
- You sign data with your private key
- Signature proves you approved it
- **Mathematically impossible to forge**

### The ECDSA Algorithm

**ECDSA (Elliptic Curve Digital Signature Algorithm)** is what Ethereum uses.

**The process:**
1. **Message:** The data you want to sign (e.g., user address)
2. **Hash:** Convert message to fixed-size hash (keccak256)
3. **Sign:** Use private key to create signature
4. **Verify:** Anyone can verify with public key (address)

**Key properties:**
- **Deterministic:** Same message + key = same signature
- **Unforgeable:** Can't create valid signature without private key
- **Verifiable:** Anyone can verify with public address
- **Non-repudiable:** Can't deny you signed it

### The Signature Components

```solidity
bytes memory signature; // 65 bytes total
```

**Breakdown:**
- **r:** 32 bytes (part of signature)
- **s:** 32 bytes (part of signature)
- **v:** 1 byte (recovery id: 27 or 28)

**Why v, r, s?** These are the mathematical components that allow signature verification and address recovery.

---

## How Signature Verification Works

### Step 1: Create the Message Hash

```solidity
bytes32 messageHash = keccak256(abi.encodePacked(msg.sender));
```

**What's happening:**
- We hash the user's address
- This creates a unique 32-byte identifier
- The signature will be bound to THIS specific address

**Why hash?** Signatures work on fixed-size data (32 bytes). Hashing ensures consistent size.

### Step 2: Add Ethereum Prefix

```solidity
bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();
```

**What this does:**
```solidity
// Adds: "\x19Ethereum Signed Message:\n32" + messageHash
```

**Why?** Security! This prevents signatures from being used on other chains or for other purposes. It's an Ethereum-specific standard (EIP-191).

### Step 3: Recover the Signer

```solidity
address signer = ethSignedMessageHash.recover(signature);
```

**The magic:** From the signature + message, we can mathematically derive the address that created it!

**How?** ECDSA math allows recovering the public key from signature, then deriving the address.

### Step 4: Verify the Signer

```solidity
require(signer == organizer, "Invalid signature");
```

**If the recovered address matches the organizer, the signature is valid!**

---

## Creating Signatures Off-Chain

### Using ethers.js

```javascript
const { ethers } = require('ethers');

// Your private key (keep secret!)
const wallet = new ethers.Wallet(privateKey);

// The address to whitelist
const userAddress = "0x123...";

// Create the message hash (same as contract)
const messageHash = ethers.utils.solidityKeccak256(
    ["address"],
    [userAddress]
);

// Sign it
const signature = await wallet.signMessage(
    ethers.utils.arrayify(messageHash)
);

console.log("Signature:", signature);
// Give this to the user!
```

### Using web3.js

```javascript
const Web3 = require('web3');
const web3 = new Web3();

// Your account
const account = web3.eth.accounts.privateKeyToAccount(privateKey);

// Create message hash
const messageHash = web3.utils.soliditySha3(
    { type: 'address', value: userAddress }
);

// Sign it
const signature = account.sign(messageHash);

console.log("Signature:", signature.signature);
```

---

## Advanced Signature Patterns

### 1. Expiring Signatures

```solidity
function enterEvent(bytes memory signature, uint256 expiration) external {
    require(block.timestamp <= expiration, "Signature expired");
    
    bytes32 messageHash = keccak256(abi.encodePacked(
        msg.sender,
        expiration
    ));
    
    bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();
    address signer = ethSignedMessageHash.recover(signature);
    
    require(signer == organizer, "Invalid signature");
    hasEntered[msg.sender] = true;
}
```

**Why?** Prevents old signatures from being reused indefinitely.

### 2. Nonce-Based Signatures

```solidity
mapping(address => uint256) public nonces;

function enterEvent(bytes memory signature, uint256 nonce) external {
    require(nonce == nonces[msg.sender], "Invalid nonce");
    
    bytes32 messageHash = keccak256(abi.encodePacked(
        msg.sender,
        nonce
    ));
    
    bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();
    address signer = ethSignedMessageHash.recover(signature);
    
    require(signer == organizer, "Invalid signature");
    
    nonces[msg.sender]++;
    hasEntered[msg.sender] = true;
}
```

**Why?** Prevents replay attacks - each signature can only be used once.

### 3. Multi-Parameter Signatures

```solidity
function mintNFT(
    uint256 tokenId,
    uint256 price,
    bytes memory signature
) external payable {
    require(msg.value == price, "Wrong price");
    
    bytes32 messageHash = keccak256(abi.encodePacked(
        msg.sender,
        tokenId,
        price
    ));
    
    bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();
    address signer = ethSignedMessageHash.recover(signature);
    
    require(signer == owner, "Invalid signature");
    
    _mint(msg.sender, tokenId);
}
```

**Why?** Encode multiple parameters in the signature for complex permissions.

---

## Security Considerations

### 1. Signature Replay Attacks

**The problem:** User uses the same signature multiple times.

**The fix:**
```solidity
mapping(bytes32 => bool) public usedSignatures;

function enterEvent(bytes memory signature) external {
    bytes32 sigHash = keccak256(signature);
    require(!usedSignatures[sigHash], "Signature already used");
    
    // ... verification ...
    
    usedSignatures[sigHash] = true;
}
```

### 2. Cross-Contract Replay

**The problem:** Signature valid on Contract A used on Contract B.

**The fix:**
```solidity
bytes32 messageHash = keccak256(abi.encodePacked(
    msg.sender,
    address(this), // Contract address
    block.chainid  // Chain ID
));
```

### 3. Signature Malleability

**The problem:** ECDSA signatures can have two valid forms.

**The fix:** OpenZeppelin's ECDSA library handles this automatically.

---

## Real-World Examples

### OpenSea Gasless Listings

**How it works:**
1. User signs listing off-chain (free)
2. Signature stored in OpenSea database
3. Buyer submits signature + payment on-chain
4. Contract verifies signature and executes trade

**Result:** Sellers pay 0 gas to list!

### EIP-2612 Permit (Gasless Approvals)

```solidity
function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
) external {
    // Verify signature
    // Set approval
}
```

**Used by:** Uniswap, Aave, Compound for gasless token approvals.

### NFT Whitelist Minting

```solidity
function whitelistMint(bytes memory signature) external payable {
    // Verify signature
    // Mint NFT
}
```

**Used by:** Most NFT projects for presale/whitelist phases.

---

## Key Concepts You've Learned

**1. Cryptographic signatures** - Proving identity without storing data

**2. ECDSA** - The algorithm Ethereum uses

**3. Signature verification** - Recovering signer from signature

**4. Off-chain signing** - Creating signatures with ethers.js/web3.js

**5. Security patterns** - Preventing replay attacks

**6. Gas optimization** - Avoiding expensive on-chain storage

---

## Why This Matters

Signatures enable:
- **Gasless transactions** - Users don't pay for certain actions
- **Scalable whitelists** - No storage costs
- **Privacy** - Whitelist not public on-chain
- **Flexibility** - Easy to update permissions
- **Meta-transactions** - Account abstraction patterns

This pattern is used in every major DeFi protocol and NFT project.

## Challenge Yourself

Extend this signature system:
1. Add expiration timestamps to signatures
2. Implement nonce-based replay protection
3. Create a multi-signature verification system
4. Build a gasless token transfer system
5. Implement EIP-2612 permit functionality

You've mastered cryptographic signatures. This is advanced Web3 development!
