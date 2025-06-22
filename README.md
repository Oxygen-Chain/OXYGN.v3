# OXYGN.v3 - Oxygen Protocol: Documentation Hub

Welcome to the Oxygen Protocol documentation center. This comprehensive collection covers all aspects of our decentralized treatment incentive system.

## 📚 Architecture Documentation


### System Status
🟢 **Production Ready**: All core components implemented with comprehensive testing  
🟢 **Cross-Chain Ready**: MockWormholeMessenger for development, official Wormhole for production  
🟢 **Security Hardened**: Fixed ownership transfer vulnerabilities, comprehensive RBAC  
🟢 **DAO Governed**: Complete governance over all system configurations  


## 🛠️ System Configuration and Operation Guide

### Table of Contents
1. [System Configuration Flow](#system-configuration-flow)
2. [Contract Initialization Sequence](#contract-initialization-sequence)
3. [Global Reserve Setup](#global-reserve-setup)
4. [System Registration Process](#system-registration-process)
5. [Reserve Creation and Configuration](#reserve-creation-and-configuration)
6. [Median Submission and Reward Process](#median-submission-and-reward-process)
7. [Reward Distribution Rules](#reward-distribution-rules)
8. [Critical Interactions](#critical-interactions)

## System Configuration Flow

### 1. Initial Contract Setup
```solidity
// 1. Deploy core contracts
OxygenToken otoken = new OxygenToken();
OxygenDAO dao = new OxygenDAO();
HealingReserves healingReserve = new HealingReserves();
HealingRewardManager rewardManager = new HealingRewardManager();

// 2. Initialize contracts
otoken.initialize(admin, oracle);
dao.initialize(IVotes(address(otoken)), timelock);
healingReserve.initialize(address(otoken), address(dao), address(rewardManager));
rewardManager.initialize(address(healingReserve));

// 3. Deploy and set up LMNTCompute
LMNTCompute taskManager = new LMNTCompute();
taskManager.initialize(address(dao), address(otoken), address(healingReserve));
healingReserve.setTaskManagerContract(address(taskManager));
```

### 2. System Type Configuration
Before systems can request rewards, the following must be configured:
```solidity
// Configure system type parameters
healingReserve.updateSystemTypePayout("sewage", 100);    // 100% payout for sewage systems
healingReserve.updateSystemTypeBaseRate("sewage", 1e18); // 1 token per flow unit
healingReserve.updateSystemTypeConfig("sewage", 1e18, 800); // 800 flow units as threshold
```

## Contract Initialization Sequence

1. **Core Contracts**
   - OxygenToken
   - OxygenDAO
   - HealingReserves
   - HealingRewardManager
   - LMNTCompute

2. **Default Contract Templates**
   ```solidity
   // Deploy templates
   defaultSystemTemplate = new TestDefaultSystem(msg.sender);
   defaultHealingReserveTemplate = new DefaultHealingReserve(msg.sender);

   // Set default contracts
   healingReserve.setDefaultContract(address(defaultHealingReserveTemplate), 1); // 1 = reserve type
   healingReserve.setDefaultContract(address(defaultSystemTemplate), 2); // 2 = system type
   ```

3. **Role Setup**
   ```solidity
   // Grant roles
   healingReserve.grantRole(healingReserve.DAO_ROLE(), admin);
   healingReserve.grantRole(healingReserve.RESERVE_MANAGER_ROLE(), admin);
   healingReserve.grantRole(healingReserve.DAO_ROLE(), daoAddress);
   ```

## Global Reserve Setup

1. **Create Global Reserve**
   ```solidity
   // Define global attributes
   IDefaultHealingReserve.Attribute[] memory globalAttributes = new IDefaultHealingReserve.Attribute[](2);
   globalAttributes[0] = IDefaultHealingReserve.Attribute({
       name: "location",
       value: "global",
       weight: 10
   });
   globalAttributes[1] = IDefaultHealingReserve.Attribute({
       name: "treatment_type",
       value: "any",
       weight: 10
   });

   // Spawn global reserve
   address globalReserve = healingReserve.spawnDefaultReserve(
       msg.sender,
       "Global Reserve",
       globalAttributes
   );
   ```

2. **Configure Global Reserve**
   ```solidity
   // Set up roles
   IDefaultHealingReserve(globalReserve).grantRole(keccak256("DEFAULT_ADMIN_ROLE"), daoAddress);
   IDefaultHealingReserve(globalReserve).grantRole(IDefaultHealingReserve(globalReserve).DAO_ROLE(), daoAddress);
   IDefaultHealingReserve(globalReserve).grantRole(IDefaultHealingReserve(globalReserve).DAO_ROLE(), healingReservesAddress);
   IDefaultHealingReserve(globalReserve).grantRole(IDefaultHealingReserve(globalReserve).RESERVE_ROLE(), msg.sender);

   // Configure contracts
   IDefaultHealingReserve(globalReserve).setHealingReservesContract(healingReservesAddress);
   IDefaultHealingReserve(globalReserve).setEnabled(true);

   // Set as global reserve
   healingReserves.setGlobalReserve(globalReserve);
   rewardManager.setGlobalReserve(globalReserve);
   ```

3. **Global Reserve Requirements**
   - Must maintain 30% minimum balance (vs 20% for regular reserves)
   - Must be enabled by default
   - Must have universal attributes for fallback matching
   - Must be set in both HealingReserves and HealingRewardManager
   - Cannot be disabled once enabled

## System Registration Process

1. **Create System**
   ```solidity
   IDefaultSystem.SystemData memory systemData = IDefaultSystem.SystemData({
       moboSerial: "TEST123",
       name: "TestSystem",
       systemType: "sewage",
       country: "TestCountry",
       state: "TestState",
       zipCode: "12345",
       model: "TestModel",
       foundation: "TestFoundation",
       manufacturer: "TestManufacturer",
       description: "TestDescription",
       expectedflow: 1000
   });

   address system = healingReserve.spawnDefaultSystem(daoAddress, systemData);
   ```

2. **Register System Wallet**
   ```solidity
   IDefaultSystem(system).setPendingSystemAddress(walletAddress, moboSerial);
   IDefaultSystem(system).approveSystemAddress(walletAddress);
   ```

3. **Enable System**
   ```solidity
   healingReserve.enableSystem(system, true);
   ```

## Reserve Creation and Configuration

1. **Create Reserve**
   ```solidity
   IDefaultHealingReserve.Attribute[] memory attributes = new IDefaultHealingReserve.Attribute[](2);
   attributes[0] = IDefaultHealingReserve.Attribute({
       name: "location",
       value: "rio_de_janeiro",
       weight: 10
   });
   attributes[1] = IDefaultHealingReserve.Attribute({
       name: "treatment_type",
       value: "sewage",
       weight: 15
   });

   address reserve = healingReserve.spawnDefaultReserve(owner, "Test Reserve", attributes);
   ```

2. **Configure Reserve**
   ```solidity
   // Enable reserve
   healingReserve.enableReserve(reserve, true);

   // Set contracts
   IDefaultHealingReserve(reserve).setHealingReservesContract(address(healingReserve));
   IDefaultHealingReserve(reserve).setOxygenToken(address(otoken));

   // Fund reserve
   otoken.approve(reserve, amount);
   IDefaultHealingReserve(reserve).receiveContributions(amount);
   ```

## Median Submission and Reward Process

1. **Submit Median**
   ```solidity
   IHealingReserves.inputMedian memory median = IHealingReserves.inputMedian({
       i_type: "sewage",
       i_value: 50,  // Input quality
       o_type: "sewage",
       o_value: 90,  // Output quality
       i_epoch: block.timestamp,
       o_epoch: block.timestamp,
       final_delta: 40,  // Quality improvement
       f_rate: 1000,     // Flow rate
       reward: 0,
       median: 0,
       latx1M: 0,
       lonx1M: 0
   });

   IDefaultSystem(system).addPotentialMedian(median);
   ```

2. **Reward Calculation**
   The reward is calculated based on:
   - Base rate for system type
   - Flow rate
   - Quality improvement
   - System performance
   - Maintenance status

3. **Reward Distribution**
   - Rewards are distributed through matching reserves
   - Reserves maintain a minimum balance (20%)
   - Shortfalls are handled through other reserves

## Reward Distribution Rules

1. **Similarity Score Calculation**
   - Attributes must match exactly (name and value)
   - "any" attribute matches everything
   - Weights are accumulated for matching attributes
   - Global reserve serves as universal fallback

2. **Reserve Selection**
   - Reserves with highest similarity scores are selected first
   - Maximum 80% of reserve balance can be allocated
   - Minimum 20% reserve balance must be maintained (30% for global reserve)
   - Global reserve used when other reserves are insufficient

3. **Timing Rules**
   - One reward per system per 24 hours
   - Median submission interval: 1 hour minimum
   - System must be enabled and not in maintenance
   - Global reserve participates in all cascading payouts

## Critical Interactions

1. **System to HealingReserves**
   - Systems must call `requestReward` to calculate rewards
   - Systems must be properly configured before rewards
   - System type must have base rate configured
   - Global reserve serves as fallback for all systems

2. **Reserve to HealingReserves**
   - Reserves must call `allocateLiquidity` to distribute rewards
   - Reserves must maintain minimum balance
   - Reserves must have proper attributes for matching
   - Global reserve must maintain higher minimum balance

3. **HealingReserves to DefaultContracts**
   - Manages lifecycle of systems and reserves
   - Sets up proper permissions and roles
   - Handles reward distribution and shortfalls
   - Coordinates with global reserve for fallback cases 

---

## 🔗 Additional Resources

### Development Resources
- **Foundry Framework**: All contracts use Foundry for compilation, testing, and deployment
- **OpenZeppelin**: Security-audited contract libraries for access control and upgradeability
- **Solady**: Gas-optimized libraries for performance-critical operations

### Security & Auditing
- **Static Analysis**: Slither configuration in `slither.config.json`
- **Linting**: Solhint rules in `.solhint.json`
- **Test Coverage**: Comprehensive test suite with fuzzing and invariant testing
- **Audit Scripts**: Automated security scanning in `run_audit.sh`

### Key Implementation Highlights
- **HealingAttributeRegistry**: Centralized attribute validation with DAO governance
- **Cross-Chain Infrastructure**: Complete Wormhole integration for Solana connectivity
- **NFT Multiplier System**: Up to 100x reward multipliers for rare NFT holders
- **Threshold Management**: Sophisticated minting controls for economic stability
- **Emergency Controls**: Pause/resume functionality with role-based access

### Contract Addresses (Development)
Deployment addresses are tracked in `deployment/deployed.json` after running deployment scripts.

---

**📝 Note**: This documentation reflects the current implementation as of the latest commit. For the most up-to-date technical specifications, refer to the contract source code in the `src/` directory.
