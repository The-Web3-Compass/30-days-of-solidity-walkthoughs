# Decentralized Governance System: Building Your First DAO

Welcome to one of the most revolutionary concepts in Web3: **Decentralized Autonomous Organizations (DAOs)**. Today, you're not just learning to code - you're learning to build systems that can govern themselves without CEOs, boards, or traditional hierarchies.

## What's Wrong with Traditional Organizations?

Think about any company or organization:
- **A CEO makes decisions** - What if they're corrupt or incompetent?
- **Shareholders vote, but it's opaque** - Can you verify the vote count?
- **Execution is manual** - Decisions take weeks or months to implement
- **Power is centralized** - A few people control everything

**DAOs flip this model:** Every token holder gets a vote. All votes are transparent and verifiable. Approved proposals execute automatically. No single point of failure.

## Real-World DAO Examples

- **MakerDAO** - Governs the DAI stablecoin (billions in TVL)
- **Uniswap DAO** - Community controls protocol upgrades
- **Nouns DAO** - Funds creative projects through daily NFT auctions
- **ConstitutionDAO** - Raised $47M to buy the US Constitution (they lost the auction, but proved the concept!)

## What We're Building

A complete governance system where:
- **Anyone can create proposals** (with a deposit to prevent spam)
- **Token holders vote** (weighted by their holdings)
- **Quorum must be met** (minimum participation required)
- **Timelock protects against attacks** (delay before execution)
- **Execution is automatic** (code enforces the decision)

Let's build it!

---

## The Complete Contract Structure

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract DecentralizedGovernance is ReentrancyGuard {
    IERC20 public governanceToken;
    uint256 public proposalCount;
    uint256 public constant VOTING_PERIOD = 3 days;
    uint256 public constant TIMELOCK_PERIOD = 2 days;
    uint256 public constant QUORUM_PERCENTAGE = 10; // 10% of total supply
    uint256 public constant PROPOSAL_DEPOSIT = 100 * 10**18; // 100 tokens
    
    struct Proposal {
        uint256 id;
        address proposer;
        string description;
        uint256 deadline;
        uint256 votesFor;
        uint256 votesAgainst;
        bool executed;
        bool cancelled;
        uint256 executionTime;
        bytes[] executionData;
        address[] executionTargets;
        mapping(address => bool) hasVoted;
    }
    
    mapping(uint256 => Proposal) public proposals;
    
    event ProposalCreated(uint256 indexed proposalId, address indexed proposer, string description);
    event Voted(uint256 indexed proposalId, address indexed voter, bool support, uint256 weight);
    event ProposalExecuted(uint256 indexed proposalId);
    event ProposalCancelled(uint256 indexed proposalId);
    
    constructor(address _governanceToken) {
        governanceToken = IERC20(_governanceToken);
    }
    
    // Functions: createProposal, vote, finalize, execute, cancel
}
```

Let's break down each component!

---

## Understanding the Data Structures

### The Governance Token

```solidity
IERC20 public governanceToken;
```

**Why do we need a token?** In a DAO, voting power is proportional to token ownership. If you hold 1% of the tokens, you get 1% of the voting power. This aligns incentives - people with more stake in the system have more say.

**Common governance tokens:**
- UNI (Uniswap)
- COMP (Compound)
- AAVE (Aave)
- MKR (Maker)

### The Proposal Struct

```solidity
struct Proposal {
    uint256 id;                      // Unique identifier
    address proposer;                // Who created it
    string description;              // What it does
    uint256 deadline;                // When voting ends
    uint256 votesFor;                // Total votes in favor
    uint256 votesAgainst;            // Total votes against
    bool executed;                   // Has it been executed?
    bool cancelled;                  // Was it cancelled?
    uint256 executionTime;           // When it can be executed (after timelock)
    bytes[] executionData;           // Function calls to make
    address[] executionTargets;      // Contracts to call
    mapping(address => bool) hasVoted; // Track who voted
}
```

**Why so complex?** A proposal isn't just a yes/no question. It's a complete action plan:
- **Description** - "Increase staking rewards by 5%"
- **Execution data** - The actual function calls to make it happen
- **Timelock** - Safety period before execution
- **Vote tracking** - Prevent double voting

---

## Creating a Proposal

```solidity
function createProposal(
    string memory _description,
    address[] memory _targets,
    bytes[] memory _data
) external returns (uint256) {
    // 1. Require deposit to prevent spam
    require(
        governanceToken.transferFrom(msg.sender, address(this), PROPOSAL_DEPOSIT),
        "Deposit failed"
    );
    
    // 2. Create the proposal
    proposalCount++;
    Proposal storage newProposal = proposals[proposalCount];
    newProposal.id = proposalCount;
    newProposal.proposer = msg.sender;
    newProposal.description = _description;
    newProposal.deadline = block.timestamp + VOTING_PERIOD;
    newProposal.executionTargets = _targets;
    newProposal.executionData = _data;
    
    emit ProposalCreated(proposalCount, msg.sender, _description);
    return proposalCount;
}
```

**Breaking this down:**

### Step 1: The Proposal Deposit

```solidity
require(
    governanceToken.transferFrom(msg.sender, address(this), PROPOSAL_DEPOSIT),
    "Deposit failed"
);
```

**Why require a deposit?** Without it, anyone could spam the DAO with thousands of joke proposals. The deposit ensures proposers have skin in the game.

**What happens to the deposit?**
- If the proposal passes → Returned to proposer
- If it fails but meets quorum → Returned to proposer
- If it's spam (no votes) → Could be slashed (burned or sent to treasury)

### Step 2: Store the Proposal

```solidity
Proposal storage newProposal = proposals[proposalCount];
```

**Why `storage`?** We're directly modifying the blockchain state, not working with a temporary copy.

### Step 3: Set the Deadline

```solidity
newProposal.deadline = block.timestamp + VOTING_PERIOD;
```

**Typical voting periods:**
- **Short (1-3 days)** - For urgent decisions
- **Medium (5-7 days)** - Standard proposals
- **Long (14+ days)** - Major protocol changes

---

## Voting on Proposals

```solidity
function vote(uint256 _proposalId, bool _support) external {
    Proposal storage proposal = proposals[_proposalId];
    
    // 1. Validation checks
    require(block.timestamp < proposal.deadline, "Voting period ended");
    require(!proposal.hasVoted[msg.sender], "Already voted");
    require(!proposal.executed, "Already executed");
    
    // 2. Get voter's token balance (voting power)
    uint256 weight = governanceToken.balanceOf(msg.sender);
    require(weight > 0, "No voting power");
    
    // 3. Record the vote
    proposal.hasVoted[msg.sender] = true;
    
    if (_support) {
        proposal.votesFor += weight;
    } else {
        proposal.votesAgainst += weight;
    }
    
    emit Voted(_proposalId, msg.sender, _support, weight);
}
```

**Key concepts:**

### Weighted Voting

```solidity
uint256 weight = governanceToken.balanceOf(msg.sender);
```

**Why weighted?** If everyone got one vote regardless of stake, someone could create 1000 wallets with 0.001 tokens each and control the DAO. Weighted voting prevents this.

**The trade-off:** Whales (large holders) have more power. This is why many DAOs use:
- **Quadratic voting** - Square root of tokens = votes
- **Delegation** - Lend your voting power to trusted representatives
- **Vote locking** - Lock tokens longer = more voting power

### Preventing Double Voting

```solidity
require(!proposal.hasVoted[msg.sender], "Already voted");
proposal.hasVoted[msg.sender] = true;
```

This is critical! Without it, someone could vote, transfer tokens to another wallet, and vote again.

---

## Finalizing and Executing Proposals

### Step 1: Finalize (Check if it Passed)

```solidity
function finalize(uint256 _proposalId) external {
    Proposal storage proposal = proposals[_proposalId];
    
    require(block.timestamp >= proposal.deadline, "Voting still active");
    require(!proposal.executed, "Already executed");
    
    // Calculate quorum
    uint256 totalSupply = governanceToken.totalSupply();
    uint256 quorumRequired = (totalSupply * QUORUM_PERCENTAGE) / 100;
    uint256 totalVotes = proposal.votesFor + proposal.votesAgainst;
    
    require(totalVotes >= quorumRequired, "Quorum not met");
    require(proposal.votesFor > proposal.votesAgainst, "Proposal rejected");
    
    // Set execution time (timelock)
    proposal.executionTime = block.timestamp + TIMELOCK_PERIOD;
}
```

**What's quorum?** The minimum participation required for a vote to be valid. If only 5 people vote out of 10,000 token holders, should that decision count? Quorum says "no."

**Common quorum levels:**
- **Low (4-10%)** - Easier to pass proposals
- **Medium (10-20%)** - Balanced approach
- **High (30%+)** - Very conservative, hard to pass anything

### Step 2: Execute (After Timelock)

```solidity
function execute(uint256 _proposalId) external nonReentrant {
    Proposal storage proposal = proposals[_proposalId];
    
    require(proposal.executionTime > 0, "Not finalized");
    require(block.timestamp >= proposal.executionTime, "Timelock active");
    require(!proposal.executed, "Already executed");
    
    proposal.executed = true;
    
    // Execute all the calls
    for (uint256 i = 0; i < proposal.executionTargets.length; i++) {
        (bool success, ) = proposal.executionTargets[i].call(proposal.executionData[i]);
        require(success, "Execution failed");
    }
    
    // Return deposit to proposer
    governanceToken.transfer(proposal.proposer, PROPOSAL_DEPOSIT);
    
    emit ProposalExecuted(_proposalId);
}
```

**Why the timelock?** Security! If a malicious proposal passes (maybe through a governance attack), the community has time to:
- Detect the issue
- Create a counter-proposal
- Exit the protocol if needed

**Famous timelock saves:**
- Compound discovered a bug in a passed proposal during the timelock
- Uniswap community caught a malicious proposal before execution

---

## Real-World Attack Vectors

### 1. Flash Loan Governance Attack

**The attack:**
1. Attacker takes a flash loan of governance tokens
2. Creates and votes on a malicious proposal
3. Returns the tokens in the same transaction

**The defense:**
```solidity
// Snapshot voting power at proposal creation
mapping(uint256 => mapping(address => uint256)) public votingPowerSnapshot;
```

### 2. Proposal Spam

**The attack:** Flood the DAO with thousands of proposals to hide malicious ones

**The defense:** Proposal deposits (which we implemented!)

### 3. Low Quorum Exploitation

**The attack:** Pass proposals when participation is low (holidays, weekends)

**The defense:** Adaptive quorum that increases if participation drops

---

## Advanced Features to Add

### 1. Delegation

```solidity
mapping(address => address) public delegates;

function delegate(address _to) external {
    delegates[msg.sender] = _to;
}

function getVotingPower(address _voter) public view returns (uint256) {
    // Count delegated votes
}
```

### 2. Proposal Cancellation

```solidity
function cancel(uint256 _proposalId) external {
    Proposal storage proposal = proposals[_proposalId];
    require(msg.sender == proposal.proposer, "Not proposer");
    require(!proposal.executed, "Already executed");
    proposal.cancelled = true;
}
```

### 3. Vote Delegation with Reason

```solidity
function voteWithReason(uint256 _proposalId, bool _support, string memory _reason) external {
    // Same as vote() but emits reason
    emit VotedWithReason(_proposalId, msg.sender, _support, weight, _reason);
}
```

---

## Key Concepts You've Learned

**1. Token-weighted voting** - Stake determines power

**2. Quorum requirements** - Minimum participation for validity

**3. Timelock mechanisms** - Safety delay before execution

**4. Proposal deposits** - Economic spam prevention

**5. On-chain execution** - Automatic implementation of decisions

**6. Governance attacks** - Flash loans, low participation exploits

---

## Why This Matters

You just built the foundation of decentralized governance. This pattern powers:
- **Protocol upgrades** - Community decides on new features
- **Treasury management** - Collective control of funds
- **Parameter changes** - Adjust fees, rewards, limits
- **Emergency actions** - Pause contracts, fix bugs

DAOs are still experimental, but they represent a fundamental shift in how humans coordinate. You're now part of that revolution.

## Challenge Yourself

Extend this DAO:
1. Add delegation (vote on behalf of others)
2. Implement quadratic voting
3. Add proposal categories (treasury, technical, social)
4. Create an emergency pause mechanism
5. Build a reputation system for active voters

You've mastered decentralized governance. This is cutting-edge Web3 development!
