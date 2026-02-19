# PollStation Contract: Mastering Arrays and Mappings

Welcome to a fundamental lesson in Solidity data structures: **arrays and mappings**. Today, you're building a voting system that demonstrates how to store and manage collections of data on-chain.

## Why Data Structures Matter

**The challenge:** You need to store multiple pieces of related data:
- List of candidates
- Vote counts for each candidate
- Who has voted
- Voting history

**Without proper data structures:**
```solidity
// This doesn't scale!
string public candidate1;
string public candidate2;
string public candidate3;
uint256 public votes1;
uint256 public votes2;
uint256 public votes3;
```

**Problems:**
- Fixed number of candidates
- Can't iterate over candidates
- Duplicate code everywhere
- Impossible to maintain

**The solution:** Arrays and mappings!

### Declaring the Candidate List and Vote Tracking

```
string[] public candidateNames;
mapping(string => uint256) voteCount;
```

#### Arrays – Storing a List of Candidates

string[] is an array of strings. It stores multiple values in an ordered list.

#### Mappings – Storing Vote Counts

mapping(string => uint256) links a candidate's name to their vote count for instant lookups.

### Adding Candidates – The addCandidateNames() Function

```
function addCandidateNames(string memory _candidateNames) public {
    candidateNames.push(_candidateNames);
    voteCount[_candidateNames] = 0;
}
```

### Retrieving the List of Candidates

```
function getcandidateNames() public view returns (string[] memory) {
    return candidateNames;
}
```

### Voting for a Candidate – The vote() Function

```
function vote(string memory _candidateNames) public {
    voteCount[_candidateNames] += 1;
}
```

### Checking a Candidate’s Votes

```
function getVote(string memory _candidateNames) public view returns (uint256) {
    return voteCount[_candidateNames];
}
```

---

## Arrays vs Mappings: When to Use Each

### Arrays - Ordered Lists

**Use when:**
- You need to iterate over items
- Order matters
- You need to know the length
- You want to access by index

**Example use cases:**
- List of candidates
- Transaction history
- Leaderboards
- Token holder lists

**Limitations:**
- Expensive to search (O(n))
- Expensive to delete from middle
- Gas costs increase with size

### Mappings - Key-Value Pairs

**Use when:**
- You need fast lookups (O(1))
- You have a unique key
- Order doesn't matter
- You don't need to iterate

**Example use cases:**
- Vote counts by candidate
- Balances by address
- Ownership records
- Access control lists

**Limitations:**
- Can't iterate over keys
- Can't get the length
- All keys exist (return default value)

---

## Security Considerations

### 1. Double Voting

**The problem:** Nothing stops someone from voting multiple times!

```solidity
// VULNERABLE
function vote(string memory _candidateNames) public {
    voteCount[_candidateNames] += 1;
}
```

**The fix:**
```solidity
mapping(address => bool) public hasVoted;

function vote(string memory _candidateNames) public {
    require(!hasVoted[msg.sender], "Already voted");
    hasVoted[msg.sender] = true;
    voteCount[_candidateNames] += 1;
}
```

### 2. Invalid Candidates

**The problem:** Can vote for candidates that don't exist!

```solidity
// VULNERABLE
vote("RandomPerson"); // Creates a vote count for non-existent candidate
```

**The fix:**
```solidity
mapping(string => bool) public isCandidate;

function addCandidateNames(string memory _candidateNames) public {
    candidateNames.push(_candidateNames);
    isCandidate[_candidateNames] = true;
    voteCount[_candidateNames] = 0;
}

function vote(string memory _candidateNames) public {
    require(isCandidate[_candidateNames], "Invalid candidate");
    require(!hasVoted[msg.sender], "Already voted");
    hasVoted[msg.sender] = true;
    voteCount[_candidateNames] += 1;
}
```

### 3. Unauthorized Candidate Addition

**The problem:** Anyone can add candidates!

**The fix:**
```solidity
address public admin;

constructor() {
    admin = msg.sender;
}

modifier onlyAdmin() {
    require(msg.sender == admin, "Not admin");
    _;
}

function addCandidateNames(string memory _candidateNames) public onlyAdmin {
    candidateNames.push(_candidateNames);
    isCandidate[_candidateNames] = true;
    voteCount[_candidateNames] = 0;
}
```

---

## Advanced Voting Features

### 1. Voting Period

```solidity
uint256 public votingStart;
uint256 public votingEnd;

constructor(uint256 _durationInDays) {
    votingStart = block.timestamp;
    votingEnd = block.timestamp + (_durationInDays * 1 days);
}

function vote(string memory _candidateNames) public {
    require(block.timestamp >= votingStart, "Voting not started");
    require(block.timestamp <= votingEnd, "Voting ended");
    // ... rest of logic
}
```

### 2. Get Winner

```solidity
function getWinner() public view returns (string memory winner, uint256 winningVoteCount) {
    require(block.timestamp > votingEnd, "Voting still active");
    
    winningVoteCount = 0;
    for (uint i = 0; i < candidateNames.length; i++) {
        uint256 votes = voteCount[candidateNames[i]];
        if (votes > winningVoteCount) {
            winningVoteCount = votes;
            winner = candidateNames[i];
        }
    }
}
```

### 3. Vote History

```solidity
mapping(address => string) public voterChoice;

function vote(string memory _candidateNames) public {
    // ... validation ...
    voterChoice[msg.sender] = _candidateNames;
    voteCount[_candidateNames] += 1;
}
```

---

## Gas Optimization Tips

### 1. Use Bytes32 Instead of String

```solidity
// EXPENSIVE
string[] public candidateNames;
mapping(string => uint256) voteCount;

// CHEAPER
bytes32[] public candidateNames;
mapping(bytes32 => uint256) voteCount;
```

**Why?** Strings are dynamic and expensive. bytes32 is fixed-size and cheaper.

### 2. Cache Array Length

```solidity
// EXPENSIVE - reads length every iteration
for (uint i = 0; i < candidateNames.length; i++) {
    // ...
}

// CHEAPER - reads length once
uint256 length = candidateNames.length;
for (uint i = 0; i < length; i++) {
    // ...
}
```

### 3. Use Events for History

```solidity
event Voted(address indexed voter, string candidate, uint256 timestamp);

function vote(string memory _candidateNames) public {
    // ... logic ...
    emit Voted(msg.sender, _candidateNames, block.timestamp);
}
```

**Why?** Events are much cheaper than storage for historical data.

---

## Real-World Voting Systems

### Snapshot (Off-Chain Voting)
- Votes stored off-chain (IPFS)
- Gas-free voting
- On-chain execution only

### Compound Governance
- Token-weighted voting
- Delegation support
- Timelock execution

### Aragon
- Modular voting apps
- Multiple voting strategies
- DAO framework

---

## Key Concepts You've Learned

**1. Arrays** - Ordered, iterable lists

**2. Mappings** - Fast key-value lookups

**3. Data structure selection** - When to use each

**4. Security patterns** - Preventing double voting, validation

**5. Gas optimization** - bytes32, caching, events

**6. Voting mechanics** - Periods, winners, history

---

## Why This Matters

Arrays and mappings are the foundation of every smart contract:
- **Token balances** - mapping(address => uint256)
- **NFT ownership** - mapping(uint256 => address)
- **Whitelist** - mapping(address => bool)
- **Transaction history** - array of structs

Mastering these data structures is essential for Solidity development.

## Challenge Yourself

Extend this voting system:
1. Add weighted voting (token-based)
2. Implement ranked-choice voting
3. Create a multi-poll system
4. Add vote delegation
5. Build a quadratic voting mechanism

You've mastered Solidity data structures. This is fundamental knowledge!
