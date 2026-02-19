# The Ether Piggy Bank: Handling Real ETH

Welcome to a critical milestone: **handling real Ether**. Today, you're learning the difference between tracking numbers and managing actual cryptocurrency - the foundation of every DeFi protocol.

## From Numbers to Money

**So far, you've built:**
- Auction systems (tracking bids)
- Treasure chests (access control)
- Balance trackers (mappings)

**But here's the thing:** All of those were just numbers in storage. No real money moved.

**Today changes everything.** You're building a contract that:
- Receives actual ETH
- Stores it securely
- Tracks ownership
- Sends it back on demand

**This is the foundation of:**
- DeFi protocols ($100B+ locked)
- Payment systems
- Savings accounts
- Investment pools

Let’s say you and a few close friends want to create a shared savings pool — a digital piggy bank. Not open to the public, just your own little club.

Everyone should be able to:

* Join the group (if approved)
* Deposit savings
* Check their balance
* And maybe even withdraw when they need it

And the best part? Eventually, this piggy bank will be able to accept real ETH. That’s right — actual Ether, stored securely on-chain.

Let’s start building.

## What all data do we need for our piggy bank?

Before we start writing functions, let’s think about this like we’re designing a system from scratch. We gather around a table and ask:

“What should this piggy bank remember?”

### 1. Who’s in charge?

Every group needs a leader — someone responsible for adding new members, and keeping things in order.

So we introduce the bank manager:

``` 
address public bankManager;
```

This is the person who deployed the contract. They’re the one with admin privileges — the only one who can approve new members into the club.

### 2. Who are the members?

We need a way to track who’s allowed to use the piggy bank.

``` 
address[] members;
mapping(address => bool) public registeredMembers;
```

Here’s why we use both:

* members: A simple list (array) of everyone in the group.
* registeredMembers: A mapping that lets the contract quickly check if an address is allowed to enter. (Remember: mappings are much faster and cheaper for checking access!)

Think of members like your group chat list. And registeredMembers like the security gate that says yes or no when someone tries to interact with the contract.

### 3. How much has each person saved?

Of course, we need to know how much each member has deposited over time.

``` 
mapping(address => uint256) balance;
```

This is the heart of the piggy bank. It stores the balance for every member.

## Setting things up: constructor

Now that we know what we need, let’s write the part that sets it all in motion.

``` 
constructor() {
    bankManager = msg.sender;
    members.push(msg.sender);
}
```

Here’s what’s happening:

* We set the bankManager to the person who deployed the contract.
* We add the manager as the first member of the piggy bank.

We’ve officially opened our piggy bank for business.

## Now let’s build the rules (Modifiers)

Before we start writing the actual features, we need to talk about who can do what.

This is where modifiers come in — little pieces of reusable logic that protect your functions.

onlyBankManager

``` 
modifier onlyBankManager() {
    require(msg.sender == bankManager, "Only bank manager can perform this action");
    _;
}
```

This modifier makes sure that only the manager can call certain functions — like adding members.

If someone else tries? The contract says: “Nope. Access denied.”

### onlyRegisteredMember

``` 
modifier onlyRegisteredMember() {
    require(registeredMembers[msg.sender], "Member not registered");
    _;
}
```

This one ensures that only people who’ve been officially added to the club can deposit or withdraw savings.

Alright — now that we have our roles and memory in place, let’s move on to the actual features.

## Adding new members

Let’s say the manager wants to add one of their friends, Alex.

Here’s the function:

``` 
function addMembers(address _member) public onlyBankManager {
    require(_member != address(0), "Invalid address");
    require(_member != msg.sender, "Bank Manager is already a member");
    require(!registeredMembers[_member], "Member already registered");

    registeredMembers[_member] = true;
    members.push(_member);
}
```

We check:

* Is the address valid?
* Is the person already a member?
* Is the manager trying to add themselves again?

Once everything looks good, we approve the new member.

### Viewing the members

Let’s say someone wants to see who all is in the group. Maybe to double-check that they’ve been added.

``` 
function getMembers() public view returns (address[] memory) {
    return members;
}
```

This is a public view function — it simply returns the full list of members.

## Depositing (simulated savings)

Now we get to the good stuff.

Let’s say you want to simulate adding 100 units to your piggy bank.

Here’s the function:

``` 
function deposit(uint256 _amount) public onlyRegisteredMember {
    require(_amount > 0, "Invalid amount");
    balance[msg.sender] += _amount;
}
```

This function:

* Checks that you’re a registered member.
* Increases your balance by the amount you specified.

At this point, we’re still not dealing with real ETH. These are just numbers — a placeholder for savings.

Think of this like a prototype. You’re testing the system before putting actual money in.

## Withdrawing savings (on paper)

You’ve saved up 100 units. Now you want to take out 30.

``` 
function withdraw(uint256 _amount) public onlyRegisteredMember {
    require(_amount > 0, "Invalid amount");
    require(balance[msg.sender] >= _amount, "Insufficient balance");
    balance[msg.sender] -= _amount;
}
```

We check:

* Are you a member?
* Do you have enough saved up to withdraw this amount?

No Ether is being transferred. This is just an internal update — we’re still in simulation mode.

## But what if we want to deposit real ETH?

Until now, we’ve only been playing with numbers.

But now your friends say:

“This is great. But what if we actually start saving real Ether? Like… actually send money into the piggy bank?”

That’s where we introduce two new concepts:

* payable: A keyword that tells a function it can receive actual ETH.
* msg.value: A special variable that tells the contract exactly how much ETH was sent along with the transaction.

## Depositing real Ether into the piggy bank

Let’s write a new version of the deposit function — one that accepts real ETH.

``` 
function depositAmountEther() public payable onlyRegisteredMember {
    require(msg.value > 0, "Invalid amount");
    balance[msg.sender] += msg.value;
}
```

Here’s what’s happening:

* We added the payable keyword to the function.
* We no longer ask for an amount in the input. Instead, we use msg.value.

So when a member calls this function and includes Ether in the transaction:

* The ETH goes directly into the contract’s balance.
* We update the user’s personal balance mapping so they get credit for it.

This is real savings now — actual ETH, tracked per user, just like a bank.

## What have we built?

* A secure system managed by an admin.
* A private group membership rules.
* Internal tracking for individual balances.
* The ability to receive and track actual Ether on-chain.

From simple logic to actual ETH management, you’ve now crossed into the real world of smart contracts.

## Where can we go from here?

You could now:

* Add a withdrawal function that actually sends the ETH back to the member’s wallet.
* Add a system for paying interest, or sharing a collective reward.

This piggy bank may have started with just you and your friends…

But now? It’s a legit on-chain system — and you built it.

Let’s keep going.

---

## Understanding Payable Functions

### What is payable?

```solidity
function deposit() public payable {
    // Can receive ETH
}

function withdraw() public {
    // Cannot receive ETH
}
```

**The payable keyword** tells Solidity: "This function can receive Ether."

**Without payable:** Transaction reverts if you send ETH.

### Understanding msg.value

```solidity
function depositAmountEther() public payable {
    require(msg.value > 0, "Must send ETH");
    balance[msg.sender] += msg.value;
}
```

**msg.value** is a special variable that contains the amount of wei sent with the transaction.

**Example:**
- User sends 1 ETH
- msg.value = 1000000000000000000 (1 ETH in wei)
- Contract credits their balance

---

## Withdrawing Real ETH

### The Withdrawal Function

```solidity
function withdrawEther(uint256 _amount) public onlyRegisteredMember {
    require(_amount > 0, "Invalid amount");
    require(balance[msg.sender] >= _amount, "Insufficient balance");
    
    balance[msg.sender] -= _amount;
    
    (bool success, ) = payable(msg.sender).call{value: _amount}("");
    require(success, "Transfer failed");
}
```

**Key points:**
1. **Check balance first** - Ensure user has enough
2. **Update state before transfer** - Checks-Effects-Interactions pattern
3. **Use call() for transfers** - Modern best practice
4. **Verify success** - Revert if transfer fails

### Why Checks-Effects-Interactions?

```solidity
// VULNERABLE (Effects after Interactions)
function badWithdraw(uint256 _amount) public {
    (bool success, ) = payable(msg.sender).call{value: _amount}("");
    balance[msg.sender] -= _amount; // Too late!
}

// SAFE (Effects before Interactions)
function goodWithdraw(uint256 _amount) public {
    balance[msg.sender] -= _amount; // Update first
    (bool success, ) = payable(msg.sender).call{value: _amount}("");
}
```

**Why?** Prevents reentrancy attacks where malicious contracts call back before state updates.

---

## Contract Balance vs User Balances

### Two Types of Balances

```solidity
// Individual user balances (in storage)
mapping(address => uint256) balance;

// Total contract balance (actual ETH held)
address(this).balance
```

**Example:**
- Alice deposits 1 ETH → balance[Alice] = 1 ETH, contract holds 1 ETH
- Bob deposits 2 ETH → balance[Bob] = 2 ETH, contract holds 3 ETH
- Contract balance: 3 ETH total
- Individual tracking: Alice (1 ETH), Bob (2 ETH)

**Why track both?**
- Contract balance: Total ETH held
- User balances: Who owns what

---

## Key Concepts You've Learned

**1. Payable functions** - Receiving ETH in contracts

**2. msg.value** - Amount of ETH sent with transaction

**3. ETH transfers** - Using call() to send ETH

**4. Checks-Effects-Interactions** - Security pattern for withdrawals

**5. Balance tracking** - Contract vs user balances

**6. Access control** - Who can deposit/withdraw

---

## Real-World Applications

### DeFi Protocols

**Aave, Compound, MakerDAO:**
- Users deposit ETH
- Earn interest over time
- Withdraw anytime

**Same pattern as our piggy bank!**

### Payment Splitters

```solidity
// Revenue sharing contract
function withdraw() public {
    uint256 share = balance[msg.sender];
    balance[msg.sender] = 0;
    payable(msg.sender).call{value: share}("");
}
```

### Savings Pools

**Group savings with goals:**
- Friends pool money
- Locked until target reached
- Distributed fairly

---

## Security Considerations

### 1. Reentrancy Protection

**Add ReentrancyGuard:**
```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract PiggyBank is ReentrancyGuard {
    function withdrawEther(uint256 _amount) public nonReentrant {
        // Safe from reentrancy
    }
}
```

### 2. Integer Overflow

**Solidity 0.8+** has automatic protection, but be careful:
```solidity
balance[msg.sender] += msg.value; // Safe in 0.8+
```

### 3. Failed Transfers

**Always check success:**
```solidity
(bool success, ) = payable(msg.sender).call{value: _amount}("");
require(success, "Transfer failed");
```

---

## Why This Matters

Handling real ETH is the foundation of:
- **DeFi** - $100B+ in protocols
- **Payments** - Cross-border transfers
- **Savings** - Interest-bearing accounts
- **Investment** - Pooled funds
- **Gaming** - In-game economies

Every major protocol handles ETH this way.

## Challenge Yourself

Extend this piggy bank:
1. Add interest calculation for long-term savers
2. Implement withdrawal limits (daily/weekly)
3. Create a group goal with locked funds
4. Add emergency withdrawal with penalty
5. Build a time-locked savings feature

You've mastered real ETH handling. This is production-ready smart contract development!
