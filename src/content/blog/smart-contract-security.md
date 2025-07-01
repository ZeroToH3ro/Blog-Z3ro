---
title: 'Smart Contract Security: Essential Patterns and Common Vulnerabilities'
description: 'Master smart contract security with real-world examples, security patterns, and tools to prevent costly exploits in your DeFi applications'
pubDate: 'June 20 2025'
heroImage: '../../assets/security.png'
---

# Smart Contract Security: Essential Patterns and Common Vulnerabilities

Smart contract security is paramount in the DeFi ecosystem where millions of dollars can be lost to a single bug. This comprehensive guide covers the most critical vulnerabilities, security patterns, and best practices every blockchain developer must know.

## The High Stakes of Smart Contract Security

Unlike traditional software, smart contract bugs can be catastrophic:
- **$600M+ lost in 2022 alone** to smart contract exploits
- **Code is immutable** once deployed to mainnet
- **Funds are irreversible** when stolen
- **Public visibility** means attackers can study your code

## Top 10 Smart Contract Vulnerabilities

### 1. Reentrancy Attacks

The most famous vulnerability, responsible for the DAO hack that led to Ethereum's hard fork.

**Vulnerable Code:**
```solidity
contract VulnerableBank {
    mapping(address => uint256) public balances;
    
    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        // Vulnerable: External call before state update
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        balances[msg.sender] -= amount; // State updated after external call
    }
}
```

**Attack Contract:**
```solidity
contract ReentrancyAttack {
    VulnerableBank public bank;
    
    constructor(address _bank) {
        bank = VulnerableBank(_bank);
    }
    
    function attack() external payable {
        bank.deposit{value: msg.value}();
        bank.withdraw(msg.value);
    }
    
    receive() external payable {
        if (address(bank).balance >= msg.value) {
            bank.withdraw(msg.value); // Recursive call
        }
    }
}
```

**Secure Implementation:**
```solidity
contract SecureBank {
    mapping(address => uint256) public balances;
    bool private locked;
    
    modifier noReentrant() {
        require(!locked, "Reentrant call");
        locked = true;
        _;
        locked = false;
    }
    
    function withdraw(uint256 amount) public noReentrant {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        // Update state before external call
        balances[msg.sender] -= amount;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### 2. Integer Overflow/Underflow

**Vulnerable Code (Solidity <0.8.0):**
```solidity
contract VulnerableToken {
    mapping(address => uint256) balances;
    
    function transfer(address to, uint256 amount) public {
        // Vulnerable to underflow
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

**Attack:**
```solidity
// If balance is 0 and we subtract 1:
// 0 - 1 = 2^256 - 1 (maximum uint256 value)
```

**Secure Implementation:**
```solidity
// Use Solidity ^0.8.0 for automatic overflow checks
contract SecureToken {
    mapping(address => uint256) balances;
    
    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount; // Automatic overflow check
        balances[to] += amount;
    }
}
```

### 3. Flash Loan Attacks

Attackers use flash loans to manipulate prices and exploit protocols.

**Vulnerable Price Oracle:**
```solidity
contract VulnerableOracle {
    IERC20 public token;
    IUniswapV2Pair public pair;
    
    function getPrice() public view returns (uint256) {
        (uint112 reserve0, uint112 reserve1,) = pair.getReserves();
        return (reserve1 * 1e18) / reserve0; // Manipulable price
    }
}
```

**Secure Price Oracle:**
```solidity
contract SecureOracle {
    IUniswapV2Pair public pair;
    uint256 public constant PERIOD = 10 minutes;
    
    struct Observation {
        uint256 timestamp;
        uint256 price0Cumulative;
        uint256 price1Cumulative;
    }
    
    Observation[] public observations;
    
    function update() external {
        (uint256 price0Cumulative, uint256 price1Cumulative,) = 
            UniswapV2OracleLibrary.currentCumulativePrices(address(pair));
            
        observations.push(Observation({
            timestamp: block.timestamp,
            price0Cumulative: price0Cumulative,
            price1Cumulative: price1Cumulative
        }));
    }
    
    function getTWAP() external view returns (uint256) {
        require(observations.length >= 2, "Need at least 2 observations");
        
        Observation memory firstObs = observations[0];
        Observation memory lastObs = observations[observations.length - 1];
        
        uint256 timeElapsed = lastObs.timestamp - firstObs.timestamp;
        require(timeElapsed >= PERIOD, "Period not elapsed");
        
        return (lastObs.price0Cumulative - firstObs.price0Cumulative) / timeElapsed;
    }
}
```

### 4. Access Control Vulnerabilities

**Vulnerable Code:**
```solidity
contract VulnerableContract {
    address public owner;
    bool public initialized;
    
    // Anyone can call this and become owner!
    function initialize(address _owner) public {
        require(!initialized, "Already initialized");
        owner = _owner;
        initialized = true;
    }
}
```

**Secure Implementation:**
```solidity
contract SecureContract {
    address public owner;
    bool public initialized;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    constructor() {
        owner = msg.sender;
        initialized = true;
    }
    
    function setOwner(address newOwner) external onlyOwner {
        require(newOwner != address(0), "Invalid address");
        owner = newOwner;
    }
}
```

### 5. Front-Running Attacks

**Vulnerable Auction:**
```solidity
contract VulnerableAuction {
    uint256 public highestBid;
    address public highestBidder;
    
    function bid() external payable {
        require(msg.value > highestBid, "Bid too low");
        
        // Vulnerable: MEV bots can front-run with higher bid
        highestBid = msg.value;
        highestBidder = msg.sender;
    }
}
```

**Secure Commit-Reveal Auction:**
```solidity
contract SecureAuction {
    struct Bid {
        bytes32 commitment;
        uint256 deposit;
        bool revealed;
    }
    
    mapping(address => Bid) public bids;
    uint256 public revealDeadline;
    uint256 public highestBid;
    address public winner;
    
    function commitBid(bytes32 commitment) external payable {
        require(block.timestamp < revealDeadline, "Commit phase ended");
        
        bids[msg.sender] = Bid({
            commitment: commitment,
            deposit: msg.value,
            revealed: false
        });
    }
    
    function revealBid(uint256 amount, uint256 nonce) external {
        require(block.timestamp >= revealDeadline, "Reveal phase not started");
        
        Bid storage bid = bids[msg.sender];
        require(!bid.revealed, "Already revealed");
        require(keccak256(abi.encodePacked(amount, nonce, msg.sender)) == bid.commitment, "Invalid reveal");
        
        bid.revealed = true;
        
        if (amount > highestBid && bid.deposit >= amount) {
            highestBid = amount;
            winner = msg.sender;
        }
    }
}
```

## Essential Security Patterns

### 1. Checks-Effects-Interactions Pattern

Always follow this order:
1. **Checks**: Validate inputs and conditions
2. **Effects**: Update contract state
3. **Interactions**: Call external contracts

```solidity
function secureFunction(uint256 amount) external {
    // 1. Checks
    require(amount > 0, "Invalid amount");
    require(balances[msg.sender] >= amount, "Insufficient balance");
    
    // 2. Effects
    balances[msg.sender] -= amount;
    totalSupply -= amount;
    
    // 3. Interactions
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");
}
```

### 2. Pull Over Push Pattern

Instead of pushing payments, let users pull them:

```solidity
contract SecurePayments {
    mapping(address => uint256) public pendingWithdrawals;
    
    function withdraw() external {
        uint256 amount = pendingWithdrawals[msg.sender];
        require(amount > 0, "Nothing to withdraw");
        
        pendingWithdrawals[msg.sender] = 0;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Withdrawal failed");
    }
    
    // Internal function to credit payments
    function _creditPayment(address recipient, uint256 amount) internal {
        pendingWithdrawals[recipient] += amount;
    }
}
```

### 3. Circuit Breaker Pattern

Add emergency stops to pause contract functionality:

```solidity
contract CircuitBreaker {
    bool public stopped = false;
    address public owner;
    
    modifier stopInEmergency {
        require(!stopped, "Contract is stopped");
        _;
    }
    
    modifier onlyOwner {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function toggleContractActive() external onlyOwner {
        stopped = !stopped;
    }
    
    function sensitiveFunction() external stopInEmergency {
        // Function logic here
    }
}
```

### 4. Rate Limiting

Prevent abuse by limiting function calls:

```solidity
contract RateLimited {
    mapping(address => uint256) public lastCall;
    uint256 public constant COOLDOWN = 1 hours;
    
    modifier rateLimited {
        require(
            block.timestamp >= lastCall[msg.sender] + COOLDOWN,
            "Rate limit exceeded"
        );
        lastCall[msg.sender] = block.timestamp;
        _;
    }
    
    function limitedFunction() external rateLimited {
        // Function logic
    }
}
```

## Security Tools and Testing

### 1. Static Analysis Tools

**Slither:**
```bash
pip install slither-analyzer
slither . --print human-summary
```

**Mythril:**
```bash
pip install mythril
myth analyze contract.sol
```

### 2. Fuzzing with Echidna

```yaml
# echidna_config.yaml
testMode: assertion
testLimit: 50000
shrinkLimit: 5000
seqLen: 100
contractAddr: "0x00a329c0648769A73afAc7F9381E08FB43dBEA72"
```

```solidity
contract TestContract {
    function echidna_balance_under_1000() public view returns (bool) {
        return address(this).balance <= 1000 ether;
    }
}
```

### 3. Formal Verification

```solidity
contract VerifiedContract {
    uint256 public balance;
    
    /// @notice Ensures balance never exceeds max supply
    function deposit(uint256 amount) external {
        require(balance + amount <= 1000000, "Exceeds max supply");
        balance += amount;
        
        // Postcondition
        assert(balance <= 1000000);
    }
}
```

## Testing Security Scenarios

```javascript
describe("Security Tests", function() {
    it("Should prevent reentrancy attacks", async function() {
        const attacker = await ReentrancyAttack.deploy(contract.address);
        
        await expect(
            attacker.attack({ value: ethers.utils.parseEther("1") })
        ).to.be.revertedWith("Reentrant call");
    });
    
    it("Should handle integer overflow", async function() {
        const maxUint = ethers.constants.MaxUint256;
        
        await expect(
            contract.add(maxUint, 1)
        ).to.be.revertedWith("Arithmetic overflow");
    });
    
    it("Should respect access controls", async function() {
        await expect(
            contract.connect(user).adminFunction()
        ).to.be.revertedWith("Not admin");
    });
});
```

## Audit Checklist

### Pre-Audit Preparation
- [ ] Code is feature-complete
- [ ] All tests passing
- [ ] Static analysis tools ran
- [ ] Documentation complete
- [ ] Known issues documented

### Code Review Areas
- [ ] Access controls
- [ ] External calls
- [ ] Integer operations
- [ ] State changes
- [ ] Gas optimization
- [ ] Upgrade mechanisms

### Post-Audit Actions
- [ ] Fix all critical/high issues
- [ ] Consider medium/low issues
- [ ] Re-run tests
- [ ] Document changes
- [ ] Consider re-audit if major changes

## Monitoring and Incident Response

### 1. On-Chain Monitoring

```javascript
// Monitor for unusual transactions
const provider = new ethers.providers.WebSocketProvider(WS_URL);

contract.on("Transfer", (from, to, amount, event) => {
    if (amount.gt(ethers.utils.parseEther("1000"))) {
        alert("Large transfer detected!");
    }
});
```

### 2. Circuit Breakers

```solidity
contract MonitoredContract {
    uint256 public constant MAX_DAILY_VOLUME = 1000000 ether;
    uint256 public dailyVolume;
    uint256 public lastResetTime;
    
    modifier volumeCheck(uint256 amount) {
        if (block.timestamp >= lastResetTime + 1 days) {
            dailyVolume = 0;
            lastResetTime = block.timestamp;
        }
        
        require(dailyVolume + amount <= MAX_DAILY_VOLUME, "Daily volume exceeded");
        dailyVolume += amount;
        _;
    }
}
```

## Emergency Procedures

### 1. Incident Response Plan

1. **Detection**: Automated monitoring alerts
2. **Assessment**: Determine severity and impact
3. **Containment**: Pause affected functions
4. **Communication**: Notify users and stakeholders
5. **Resolution**: Fix and redeploy if possible
6. **Recovery**: Resume normal operations
7. **Lessons Learned**: Update security practices

### 2. Upgrade Strategies

```solidity
// Proxy pattern for upgradeable contracts
contract Proxy {
    address public implementation;
    address public admin;
    
    modifier onlyAdmin() {
        require(msg.sender == admin, "Not admin");
        _;
    }
    
    function upgrade(address newImplementation) external onlyAdmin {
        implementation = newImplementation;
    }
    
    fallback() external payable {
        (bool success, bytes memory data) = implementation.delegatecall(msg.data);
        require(success, "Delegatecall failed");
        
        assembly {
            return(add(data, 0x20), mload(data))
        }
    }
}
```

## Conclusion

Smart contract security is an ongoing process that requires:

1. **Understanding common vulnerabilities** and how to prevent them
2. **Following security patterns** like checks-effects-interactions
3. **Using automated tools** for testing and analysis
4. **Conducting thorough audits** before mainnet deployment
5. **Implementing monitoring** and incident response procedures

The DeFi ecosystem has lost billions to security flaws, but these losses have taught valuable lessons. By following these practices and staying updated on emerging threats, you can build more secure and robust smart contracts.

Remember: **Security is not a feature to add later—it must be built in from the ground up.**

---

*Next: Advanced testing strategies and formal verification techniques for mission-critical smart contracts.*
