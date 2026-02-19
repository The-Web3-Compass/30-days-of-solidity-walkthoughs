# Simple IOU Contract: Building On-Chain Debt Tracking

Welcome to a practical real-world application: **tracking debts between friends**. Today, you're building a smart contract that solves the age-old problem of "who owes whom" - with complete transparency and zero disputes.

## The Split Bill Problem

**The scenario:**
- You and 5 friends go out for dinner
- Bill is $300, you pay it all
- Everyone says "I'll Venmo you later"
- 2 weeks pass...
- "Wait, did I already pay you back?"

**The problems:**
- Memory fades
- No clear record
- Awkward to remind people
- Disputes arise
- Friendships strained

**The Web3 solution:** Put it on the blockchain. Immutable, transparent, indisputable.

## What We're Building

A group ledger system with:
- **Friend registry** - Only approved members can participate
- **Debt tracking** - Crystal-clear record of who owes whom
- **Shared wallet** - Deposit ETH for easy settlement
- **Instant payments** - Pay debts with one transaction
- **Multiple transfer methods** - Learn transfer() vs call()

**Real-world use cases:**
- Friend groups splitting expenses
- Roommate rent/utilities tracking
- Travel expense management
- Group gift coordination
- Small business IOUs

## State Variables: What are we keeping track of?

Before any logic can run, the contract needs to remember a few things.

### owner

``` 
address public owner;
```

The owner is the person who deployed the contract. This person will have special permissions, like adding new friends to the group. It’s like the first person who creates the Splitwise group — they manage the access.

### registeredFriends and friendList

``` 
mapping(address => bool) public registeredFriends;
address[] public friendList;
```

We want this contract to be private to your group. Not just anyone on the internet should be able to join and start logging IOUs.

Here’s how we manage that:

* registeredFriends: A mapping to check if an address is part of the group.
* friendList: An array to keep track of everyone’s address so we can list them out.

When a friend is added, their address goes into both.

### balances

``` 
mapping(address => uint256) public balances;
```

Now let’s handle the money. Each person can deposit ETH into their personal balance within the contract. That ETH can later be used to pay off debts or sent to other friends.

### debts

``` 
mapping(address => mapping(address => uint256)) public debts;
```

Here’s where the real IOU magic happens. This is a nested mapping, meaning:

``` 
debts[debtor][creditor] = amount;
```

For example:

``` 
debts[0xAsha][0xRavi] = 1.5 ether;
```

Means Asha owes Ravi 1.5 ETH. And it’s right there, stored transparently on-chain. No miscommunication, no forgetting.

## The Constructor

``` 
constructor() {
    owner = msg.sender;
    registeredFriends[msg.sender] = true;
    friendList.push(msg.sender);
}
```

When the contract is deployed: The person who deploys it (msg.sender) becomes the owner, is automatically registered as a friend, and is added to the list.

## Modifiers: Controlling Access

### onlyOwner

``` 
modifier onlyOwner() {
    require(msg.sender == owner, "Only owner can perform this action");
    _;
}
```

Ensures that only the owner can perform administrative actions like adding a friend.

### onlyRegistered

``` 
modifier onlyRegistered() {
    require(registeredFriends[msg.sender], "You are not registered");
    _;
}
```

Only registered friends can use the contract's functions.

## Functions: What Can Friends Do?

### addFriend

``` 
function addFriend(address _friend) public onlyOwner {
    require(_friend != address(0), "Invalid address");
    require(!registeredFriends[_friend], "Friend already registered");

    registeredFriends[_friend] = true;
    friendList.push(_friend);
}
```

### depositIntoWallet

``` 
function depositIntoWallet() public payable onlyRegistered {
    require(msg.value > 0, "Must send ETH");
    balances[msg.sender] += msg.value;
}
```

How a user adds ETH to their in-app balance.

### recordDebt

``` 
function recordDebt(address _debtor, uint256 _amount) public onlyRegistered {
    require(_debtor != address(0), "Invalid address");
    require(registeredFriends[_debtor], "Address not registered");
    require(_amount > 0, "Amount must be greater than 0");

    debts[_debtor][msg.sender] += _amount;
}
```

If your friend owes you money, you call this to record it.

### payFromWallet

``` 
function payFromWallet(address _creditor, uint256 _amount) public onlyRegistered {
    require(_creditor != address(0), "Invalid address");
    require(registeredFriends[_creditor], "Creditor not registered");
    require(_amount > 0, "Amount must be greater than 0");
    require(debts[msg.sender][_creditor] >= _amount, "Debt amount incorrect");
    require(balances[msg.sender] >= _amount, "Insufficient balance");

    balances[msg.sender] -= _amount;
    balances[_creditor] += _amount;
    debts[msg.sender][_creditor] -= _amount;
}
```

Pay someone back using your ETH balance stored in the contract.

### transferEther — Sending ETH Using transfer()

```
function transferEther(address payable _to, uint256 _amount) public onlyRegistered {
    require(_to != address(0), "Invalid address");
    require(registeredFriends[_to], "Recipient not registered");
    require(balances[msg.sender] >= _amount, "Insufficient balance");

    balances[msg.sender] -= _amount;
    _to.transfer(_amount);
    balances[_to] += _amount;
}
```

### transferEtherViaCall — Sending ETH Using call()

```
function transferEtherViaCall(address payable _to, uint256 _amount) public onlyRegistered {
    require(_to != address(0), "Invalid address");
    require(registeredFriends[_to], "Recipient not registered");
    require(balances[msg.sender] >= _amount, "Insufficient balance");

    balances[msg.sender] -= _amount;

    (bool success, ) = _to.call{value: _amount}("");
    balances[_to] += _amount;
    require(success, "Transfer failed");
}
```

### withdraw

``` 
function withdraw(uint256 _amount) public onlyRegistered {
    require(balances[msg.sender] >= _amount, "Insufficient balance");

    balances[msg.sender] -= _amount;

    (bool success, ) = payable(msg.sender).call{value: _amount}("");
    require(success, "Withdrawal failed");
}
```

Take your ETH out of the contract.

### checkBalance

``` 
function checkBalance() public view onlyRegistered returns (uint256) {
    return balances[msg.sender];
}
```

---

## Understanding Nested Mappings

```solidity
mapping(address => mapping(address => uint256)) public debts;
```

**This is a mapping of mappings!**

**Think of it as a spreadsheet:**
```
           Ravi    Asha    Bob
Ravi       0       50      0
Asha       0       0       100
Bob        30      0       0
```

**Reading:** `debts[Bob][Ravi] = 30` means "Bob owes Ravi 30 ETH"

**Why nested?** We need to track debts between every pair of people.

---

## Transfer vs Call: Understanding ETH Transfers

### Method 1: transfer()

```solidity
_to.transfer(_amount);
```

**Characteristics:**
- Fixed 2300 gas stipend
- Reverts on failure
- Simple and safe
- **Problem:** Can fail if recipient is a contract with expensive fallback

### Method 2: call()

```solidity
(bool success, ) = _to.call{value: _amount}("");
require(success, "Transfer failed");
```

**Characteristics:**
- Forwards all available gas
- Returns success boolean
- More flexible
- **Recommended:** Modern best practice

**Why call() is better:**
- Works with contracts that need more gas
- Gives you control over failure handling
- Future-proof (EIP-1884 broke transfer())

---

## Key Concepts You've Learned

**1. Nested mappings** - Tracking relationships between multiple entities

**2. Access control** - onlyOwner and onlyRegistered modifiers

**3. ETH handling** - Depositing, withdrawing, transferring

**4. Debt tracking** - Recording and settling IOUs

**5. Transfer methods** - transfer() vs call() patterns

**6. Group management** - Registry and permissions

---

## Security Considerations

### 1. Reentrancy Protection

**Current code is vulnerable:**
```solidity
function withdraw(uint256 _amount) public onlyRegistered {
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] -= _amount;
    (bool success, ) = payable(msg.sender).call{value: _amount}("");
    require(success);
}
```

**Why safe?** State updated BEFORE external call (Checks-Effects-Interactions pattern).

### 2. Integer Underflow

**Solidity 0.8+** has automatic overflow/underflow protection, but be careful:
```solidity
balances[msg.sender] -= _amount; // Will revert if insufficient
```

### 3. Access Control

**Only registered friends can:**
- Record debts
- Make payments
- Withdraw funds

**Only owner can:**
- Add new friends

---

## Real-World Applications

### Splitwise On-Chain

**Traditional Splitwise:**
- Centralized database
- Trust the company
- Can be shut down

**Blockchain version:**
- Decentralized
- Trustless
- Permanent record

### DAO Treasury Management

**Use case:** Track internal loans between DAO members
```solidity
// DAO member borrows from treasury
recordDebt(daoMember, 1000 ether);

// Member pays back
payFromWallet(daoTreasury, 1000 ether);
```

### Business Invoicing

**Use case:** Small business tracking customer IOUs
```solidity
// Customer owes for service
recordDebt(customer, serviceAmount);

// Customer pays
payFromWallet(business, serviceAmount);
```

---

## Challenge Yourself

Extend this IOU system:
1. Add interest on overdue debts
2. Implement debt forgiveness functionality
3. Create a dispute resolution mechanism
4. Add multi-currency support (different ERC20 tokens)
5. Build a reputation system based on payment history

You've mastered debt tracking and group financial management on-chain!
