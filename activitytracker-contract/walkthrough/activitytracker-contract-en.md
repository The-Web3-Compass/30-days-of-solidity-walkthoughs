# ActivityTracker Contract: Your First Fitness dApp

Hey there! Today we're building something practical and fun: a fitness tracker on the blockchain. But more importantly, you're going to learn about **events** - one of the most powerful features in Solidity that makes your smart contracts actually useful in the real world.

## Why Events Matter

Think about any app you use - Instagram, Strava, your banking app. They all have one thing in common: they notify you when something happens. "Your friend liked your post." "You completed a 5K run." "Payment received."

Smart contracts can do the same thing with **events**. Events are like announcements that your contract broadcasts to the world. Front-end applications listen to these events and update the UI in real-time. Without events, your dApp would be blind - it wouldn't know when things happen on-chain.

## What We're Building

A fitness tracker where users can:
- Register with their name and weight
- Log workouts (running, cycling, swimming, etc.)
- Track their progress over time
- Get milestone notifications (10 workouts, 100km total distance, weight goals)

Let's dive in!

---

## Setting Up Our Data Structures

First, we need to organize our data. We'll use **structs** - think of them as custom data types that group related information together.

```solidity
struct UserProfile {
    string name;
    uint256 weight; // in kg
    bool isRegistered;
}

struct WorkoutActivity {
    string activityType;  // "Running", "Cycling", etc.
    uint256 duration;     // in seconds
    uint256 distance;     // in meters
    uint256 timestamp;    // when the workout happened
}
```

**Why structs?** Instead of having separate variables for name, weight, and registration status, we bundle them together. It's cleaner and makes our code easier to read.

Now let's store this data:

```solidity
mapping(address => UserProfile) public userProfiles;
mapping(address => WorkoutActivity[]) private workoutHistory;
mapping(address => uint256) public totalWorkouts;
mapping(address => uint256) public totalDistance;
```

**Breaking this down:**
- `userProfiles` - Each wallet address gets one profile
- `workoutHistory` - Each user has an array of all their workouts
- `totalWorkouts` - Quick counter for how many workouts someone has done
- `totalDistance` - Running total of all distance covered

Notice `workoutHistory` is `private`? That's because it could get huge. We don't want to expose the entire array publicly (gas costs would be insane). Instead, we'll provide specific functions to query it.

---

## Declaring Events: Broadcasting to the World

Here's where the magic happens. We declare events that will be emitted when important things occur:

```solidity
event UserRegistered(address indexed userAddress, string name, uint256 timestamp);
event ProfileUpdated(address indexed userAddress, uint256 newWeight, uint256 timestamp);
event WorkoutLogged(address indexed userAddress, string activityType, uint256 duration, uint256 distance, uint256 timestamp);
event MilestoneAchieved(address indexed userAddress, string milestone, uint256 timestamp);
```

**What's that `indexed` keyword?** It makes the parameter searchable. Your front-end can filter events by specific addresses. For example, "Show me all workouts logged by user 0x123..."

Events are cheap to emit and stored in transaction logs (not in contract storage), making them perfect for tracking historical data.

---

## Function 1: Registering a User

```solidity
function registerUser(string memory _name, uint256 _weight) public {
    require(!userProfiles[msg.sender].isRegistered, "User already registered");

    userProfiles[msg.sender] = UserProfile({
        name: _name,
        weight: _weight,
        isRegistered: true
    });

    emit UserRegistered(msg.sender, _name, block.timestamp);
}
```

**What's happening here?**

1. **Check if already registered** - We don't want duplicate registrations
2. **Create the profile** - Store the user's data in our mapping
3. **Emit an event** - Broadcast to the world that a new user joined!

The `emit` keyword is how we trigger events. Any app listening will immediately know someone registered and can update the UI: "Welcome, Alice! Your profile is ready."

---

## Function 2: Updating Weight

```solidity
function updateWeight(uint256 _newWeight) public onlyRegistered {
    UserProfile storage profile = userProfiles[msg.sender];
    
    // Check if they lost 5% or more of their weight
    if (_newWeight < profile.weight && (profile.weight - _newWeight) * 100 / profile.weight >= 5) {
        emit MilestoneAchieved(msg.sender, "Weight Goal Reached", block.timestamp);
    }
    
    profile.weight = _newWeight;
    emit ProfileUpdated(msg.sender, _newWeight, block.timestamp);
}
```

**Key concepts:**

- **`storage` keyword** - We're directly modifying the stored profile, not a copy
- **Milestone detection** - If weight drops by 5% or more, we celebrate with an event!
- **Always emit** - Even simple updates get an event so the UI stays in sync

Notice we use `onlyRegistered` modifier (you'd define this to check `isRegistered`). This prevents unregistered users from calling this function.

---

## Function 3: Logging a Workout

This is the heart of our contract:

```solidity
function logWorkout(string memory _activityType, uint256 _duration, uint256 _distance) public onlyRegistered {
    // Create the workout record
    WorkoutActivity memory newWorkout = WorkoutActivity({
        activityType: _activityType,
        duration: _duration,
        distance: _distance,
        timestamp: block.timestamp
    });

    // Store it and update counters
    workoutHistory[msg.sender].push(newWorkout);
    totalWorkouts[msg.sender]++;
    totalDistance[msg.sender] += _distance;

    // Broadcast the workout
    emit WorkoutLogged(msg.sender, _activityType, _duration, _distance, block.timestamp);

    // Check for workout count milestones
    if (totalWorkouts[msg.sender] == 10) {
        emit MilestoneAchieved(msg.sender, "10 Workouts Completed", block.timestamp);
    } else if (totalWorkouts[msg.sender] == 50) {
        emit MilestoneAchieved(msg.sender, "50 Workouts Completed", block.timestamp);
    }

    // Check for distance milestone (100km = 100,000 meters)
    if (totalDistance[msg.sender] >= 100000 && totalDistance[msg.sender] - _distance < 100000) {
        emit MilestoneAchieved(msg.sender, "100K Total Distance", block.timestamp);
    }
}
```

**What's cool here:**

1. **We create a workout record** - Using `memory` because it's temporary (until we push it to storage)
2. **Update multiple trackers** - History array, workout count, total distance
3. **Emit the workout event** - Front-end can show "Alice just ran 5km!"
4. **Auto-detect milestones** - When you hit 10 workouts or 100km, the contract celebrates with you

That last distance check is clever: `totalDistance[msg.sender] >= 100000 && totalDistance[msg.sender] - _distance < 100000` means "we just crossed the 100km threshold with this workout."

---

## Why This Matters

You just built a contract that:
- **Stores data efficiently** using structs and mappings
- **Broadcasts events** that front-ends can listen to
- **Tracks progress** and celebrates achievements
- **Scales well** - Each user's data is isolated

Events are what make blockchain apps feel alive. Without them, you'd have to constantly poll the blockchain asking "did anything change?" With events, your app knows instantly.

## What's Next?

Try adding:
- A function to retrieve workout history (with pagination!)
- More milestone types (fastest 5K, most workouts in a week)
- Social features (follow friends, compare stats)
- NFT badges for achievements

You've mastered events. This is a huge step in your Solidity journey!
