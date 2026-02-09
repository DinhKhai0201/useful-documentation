# Giáo Trình Tokenomics & Triển Khai Hệ Thống Token
## Dành cho Dev Contract Thiếu Kinh Nghiệm Kinh Tế

**Tác giả**: Perplexity AI  
**Ngày**: 09/02/2026  
**Mục đích**: Cung cấp framework đầy đủ từ kiến thức kinh tế cơ bản đến triển khai smart contract thực tế cho hệ thống token DeFi/Perp DEX.

---

## MỤC LỤC

### **PHẦN 1: KIẾN THỨC KINH TẾ CƠ BẢN (FOUNDATION)**
1. [Tokenomics Là Gì?](#1-tokenomics-là-gì)
2. [6 Trụ Cột Tokenomics](#2-6-trụ-cột-tokenomics)
3. [Supply Models: Fixed vs Dynamic vs Deflationary](#3-supply-models)
4. [Token Utility: Tại Sao User Phải Cần Token?](#4-token-utility)

### **PHẦN 2: PHÂN PHỐI & VESTING (DISTRIBUTION)**
5. [Token Allocation: Chia Cho Ai?](#5-token-allocation)
6. [Vesting Schedule: Cliff, Linear, Milestone](#6-vesting-schedule)
7. [Token Generation Event (TGE)](#7-token-generation-event)

### **PHẦN 3: REWARD SYSTEMS (INCENTIVE DESIGN)**
8. [Staking & LP Rewards](#8-staking--lp-rewards)
9. [Emissions Schedule & Taper](#9-emissions-schedule--taper)
10. [Fee Distribution Models](#10-fee-distribution-models)
11. [Buyback & Burn Mechanism](#11-buyback--burn-mechanism)

### **PHẦN 4: GOVERNANCE & VALUE ACCRUAL**
12. [Governance Models (veToken, Quadratic)](#12-governance-models)
13. [Value Accrual: Làm Sao Token Tăng Giá?](#13-value-accrual)

### **PHẦN 5: TRIỂN KHAI SMART CONTRACT**
14. [Contract Architecture Overview](#14-contract-architecture-overview)
15. [ERC20 + Non-Transferable esToken](#15-erc20--non-transferable-estoken)
16. [Staking Rewards Contract](#16-staking-rewards-contract)
17. [Vesting Contract](#17-vesting-contract)
18. [Fee Router & Buyback Contract](#18-fee-router--buyback-contract)

### **PHẦN 6: LAUNCH STRATEGY & LONG-TERM PLAN**
19. [Bootstrap Liquidity (Pre-Launch)](#19-bootstrap-liquidity)
20. [Go-to-Market Plan (First 6 Months)](#20-go-to-market-plan)
21. [Metrics Tracking & KPI](#21-metrics-tracking--kpi)
22. [Common Pitfalls & How to Avoid](#22-common-pitfalls--how-to-avoid)

### **PHẦN 7: CASE STUDIES**
23. [GMX Tokenomics Deep Dive](#23-gmx-tokenomics-deep-dive)
24. [Curve veToken Model](#24-curve-vetoken-model)
25. [MegaBanX Example Implementation](#25-megabanx-example-implementation)

---

## PHẦN 1: KIẾN THỨC KINH TẾ CƠ BẢN

### 1. Tokenomics Là Gì?

**Định nghĩa**: Tokenomics = Token + Economics, là thiết kế kinh tế của một token/coin, bao gồm:
- **Supply** (cung): Bao nhiêu token tồn tại?
- **Distribution** (phân phối): Token đến tay ai, khi nào?
- **Utility** (tiện ích): Token dùng để làm gì?
- **Incentives** (động lực): Tại sao user muốn giữ/mua token?
- **Value accrual** (tăng giá trị): Làm sao token tăng giá theo thời gian?

**Ví dụ đơn giản**:
```
Bitcoin = Fixed supply (21M), Mining rewards giảm dần, Utility = store of value
Ethereum = Unlimited supply → EIP-1559 burn → Deflationary, Utility = gas fees
MBX (của bạn) = Fixed 100M supply, Staking rewards, Utility = fee discount + governance
```

**Tại sao quan trọng**:
- Tokenomics tốt → Token price tăng, community loyal, protocol sustainable.
- Tokenomics tệ → Token dump, mercenary capital rút, project die.
- **95% tokens crash sau 3 tháng TGE** vì tokenomics poorly designed.

---

### 2. 6 Trụ Cột Tokenomics

Theo framework của BlackTokenomics & ThreeSigma (2025-2026), mọi tokenomics design phải cover 6 verticals:

#### **2.1. Purpose & Utility**
**Token dùng để làm gì?**
- ✅ Governance (vote proposals)
- ✅ Staking (earn rewards)
- ✅ Fee discount (trading cheaper)
- ✅ Collateral (borrow against it)
- ✅ LP provision (provide liquidity)
- ✅ Access (premium features)

**Câu hỏi critical**:
- Nếu không có token, protocol có hoạt động được không?
- User có động lực giữ token lâu dài không?

#### **2.2. Economic Model**
**Token supply & emissions**:
- Fixed supply (Bitcoin-style)?
- Inflationary (Ethereum pre-merge)?
- Deflationary (burn > mint)?
- Hybrid (mint + burn balanced)?

#### **2.3. Distribution**
**Allocation pie chart**:
```
Example:
- Team/Advisors: 20% (vesting 4 năm)
- Investors: 15% (vesting 2 năm)
- Treasury/Ecosystem: 30%
- Community/Rewards: 25%
- Liquidity/DEX: 10%
```

#### **2.4. Value Creation & Accrual**
**Làm sao token capture value từ protocol?**
- Fee share (GMX model)
- Buyback & burn (BNB model)
- veToken boost (Curve model)
- Revenue share (Real yield)

#### **2.5. Governance**
**Ai decide protocol changes?**
- 1 token = 1 vote (basic)
- veToken (lock longer = vote nhiều hơn, Curve-style)
- Quadratic voting (reduce whale power)
- Delegated (AAVE-style)

#### **2.6. Fundraising**
**Raise bao nhiêu? Với điều khoản gì?**
- Seed: $500K @ $0.01/token (50M tokens), vesting 2 năm
- Private: $2M @ $0.05/token (40M tokens), vesting 18 tháng
- Public: $1M @ $0.1/token (10M tokens), no vesting

---

### 3. Supply Models

#### **3.1. Fixed Supply**
```
Total = 100M tokens (never change)
Example: Bitcoin (21M), MBX (100M)
```
**Pros**:
- Scarcity rõ ràng
- Anti-inflation
- Easy marketing ("only 100M exist")

**Cons**:
- Rewards phải từ fees (khi emissions hết)
- Không linh hoạt adjust

**Khi nào dùng**: Store of value, governance token, long-term hold.

#### **3.2. Inflationary (Unlimited Supply)**
```
Total = Unlimited, mint new token mỗi block/epoch
Example: ETH (pre-merge), DOGE
```
**Pros**:
- Linh hoạt tạo rewards
- Bootstrap early users dễ

**Cons**:
- Sell pressure lớn
- Token dump nếu không balance

**Khi nào dùng**: Utility token cần liquidity cao, gaming/metaverse rewards.

#### **3.3. Deflationary (Burn > Mint)**
```
Total = Giảm dần theo thời gian
Example: BNB (burn quarterly), SHIB (burn campaigns)
```
**Pros**:
- Price support bởi scarcity
- Marketing-friendly

**Cons**:
- Phải có volume fees đủ lớn
- Mechanism phức tạp

**Khi nào dùng**: Exchange tokens, DeFi protocols với high volume.

#### **3.4. Hybrid (Cân Bằng Mint + Burn)**
```
Mint: Staking rewards (2% APY)
Burn: 10% fees weekly
Net: Deflationary nếu fees > emissions
Example: MegaBanX MBX, GMX (partial)
```
**Best practice cho DeFi 2026**: Hybrid với emissions taper.

---

### 4. Token Utility

**Rule #1**: Token phải có **real utility**, không chỉ speculative.

#### **4.1. Utility Types Ranking**

| Utility | Strength | Example |
|---------|----------|---------|
| **Fee discount** | ⭐⭐⭐ | Stake MBX → 15% discount trading fees |
| **Governance** | ⭐⭐ | Vote proposal, parameter changes |
| **Revenue share** | ⭐⭐⭐⭐⭐ | Stake earn 30% protocol fees (real yield) |
| **Collateral** | ⭐⭐⭐⭐ | Borrow stablecoin against MBX |
| **Access** | ⭐⭐ | Hold 1000 MBX → unlock premium features |
| **LP provision** | ⭐⭐⭐ | Provide MBX/USDC liquidity earn fees |

#### **4.2. Designing Utility**

**Framework checklist**:
- [ ] User phải dùng token để tương tác protocol?
- [ ] Holding token có reward rõ ràng (fees, discount, yield)?
- [ ] Utility tăng theo ecosystem growth (network effect)?
- [ ] User có lý do lock/stake token lâu dài?

**Example MegaBanX MBX**:
```
✅ Fee discount: 0-15% tùy lock duration
✅ Revenue share: 30% protocol fees → stakers
✅ Governance: Vote fee splits, new markets
✅ Collateral: (future) Borrow USDC against MBX
✅ LP rewards: Provide MBX/USDC earn extra APR
```

---

## PHẦN 2: PHÂN PHỐI & VESTING

### 5. Token Allocation

**Standard allocation cho DeFi protocol** (2026 best practices):

#### **5.1. Pie Chart Template**

```
Total Supply: 100,000,000 MBX

├─ Team & Advisors: 20,000,000 (20%)
│  └─ Vesting: 4 years, 1 year cliff
│
├─ Early Investors: 15,000,000 (15%)
│  ├─ Seed: 5M @ $0.01 (vesting 24 months)
│  └─ Private: 10M @ $0.05 (vesting 18 months)
│
├─ Treasury/DAO: 25,000,000 (25%)
│  └─ Controlled by governance, no vesting
│
├─ Ecosystem/Rewards: 30,000,000 (30%)
│  ├─ Staking rewards: 20M (emissions 12 tháng)
│  └─ LP incentives: 10M (emissions 6 tháng)
│
├─ Public Sale/IDO: 5,000,000 (5%)
│  └─ No vesting (instant liquidity)
│
└─ Liquidity (DEX): 5,000,000 (5%)
   └─ Locked in Uniswap/MegaETH AMM
```

#### **5.2. Key Principles**

1. **Team ≤ 20%**: Nhiều hơn = red flag cho investors.
2. **Community ≥ 50%** (Ecosystem + Rewards + Public + Liquidity).
3. **Vesting cho insiders**: Team, advisors, investors MUST vest.
4. **Treasury flexible**: DAO control, no hard vesting.

#### **5.3. Red Flags**

❌ Team allocation >30%  
❌ Không có vesting cho founders  
❌ Public sale <3% (too centralized)  
❌ Treasury >40% (team có thể dump)

---

### 6. Vesting Schedule

**Vesting = Release token dần dần theo thời gian**, chống dump.

#### **6.1. Core Components**

**Cliff Period**:
```
Lock token hoàn toàn, 0 token unlock.
Duration: 6-12 tháng thường (founders), 1-3 tháng (investors).
Purpose: Chứng minh long-term commitment.
```

**Vesting Duration**:
```
Tổng thời gian unlock hết token.
Team/Advisors: 3-4 năm
Investors: 18-24 tháng
Community: 6-12 tháng
```

**Release Frequency**:
```
Monthly: Unlock mỗi tháng (common)
Quarterly: Mỗi 3 tháng (less volatile)
Daily: Unlock mỗi ngày (smoothest)
```

#### **6.2. Vesting Types**

**Linear Vesting** (Most Common):
```
Example: 10M tokens, vesting 24 tháng, monthly
→ Unlock 416,666 tokens/tháng (10M / 24)

Timeline:
Month 0-12 (cliff): 0 tokens
Month 13: 416,666 tokens
Month 14: 416,666 tokens
...
Month 36: 416,666 tokens (total 10M)
```

**Cliff-Based**:
```
Example: 12 tháng cliff, sau đó unlock 100%
→ Month 0-11: 0
→ Month 12: 10M tokens at once
```
⚠️ Risky: Dump risk cao, ít dùng cho insiders.

**Graded (Tăng Dần)**:
```
Year 1: 10% unlock
Year 2: 20% unlock
Year 3: 30% unlock
Year 4: 40% unlock
→ Align với project growth
```

**Milestone-Based**:
```
TVL $100M → unlock 25%
Volume $1B → unlock 25%
Mainnet launch → unlock 25%
DAO transition → unlock 25%
→ Tie vesting với performance
```

**Hybrid (Recommended 2026)**:
```
Team: 12 tháng cliff + 36 tháng linear monthly
Investors: 6 tháng cliff + 18 tháng linear monthly
Community: No cliff + 12 tháng linear weekly
```

#### **6.3. Vesting Formula**

```solidity
// Linear vesting formula
function claimable(address user) public view returns (uint256) {
    VestingSchedule memory schedule = schedules[user];
    
    // Before cliff
    if (block.timestamp < schedule.startTime + schedule.cliff) {
        return 0;
    }
    
    // After vesting end
    if (block.timestamp >= schedule.startTime + schedule.duration) {
        return schedule.totalAmount - schedule.claimed;
    }
    
    // During vesting
    uint256 elapsed = block.timestamp - schedule.startTime;
    uint256 vested = (schedule.totalAmount * elapsed) / schedule.duration;
    return vested - schedule.claimed;
}
```

---

### 7. Token Generation Event (TGE)

**TGE = Ngày token được mint lần đầu** (Genesis event).

#### **7.1. TGE Unlock Breakdown**

**Chuẩn conservative** (chống dump):
```
Total circulating at TGE: 15-20% max

Breakdown:
├─ Public sale: 5% (instant liquid)
├─ DEX liquidity: 5% (locked in AMM)
├─ Ecosystem kickstart: 5% (initial rewards)
└─ Marketing/airdrops: 2%

Locked until cliff ends:
└─ Team, Investors, Treasury: 80%
```

**Example timeline**:
```
Day 0 (TGE):
- Mint 100M MBX total supply
- Unlock 15M to circulation
- Lock 85M in vesting contracts

Month 6:
- Investor cliff ends → monthly unlock starts
- Circulating: 20M

Month 12:
- Team cliff ends → monthly unlock starts
- Emissions tapers
- Circulating: 35M

Month 24:
- Most vesting done
- Circulating: 70M

Month 48:
- All vesting complete
- Circulating: 100M (minus burns)
```

#### **7.2. TGE Best Practices**

✅ Circulating at TGE: 10-20%  
✅ Team cliff ≥ 12 months  
✅ Liquidity locked (Uniswap V2 lock)  
✅ Vesting contracts audited  
✅ Transparent tokenomics doc published  

❌ >30% circulating at TGE (dump risk)  
❌ No cliff for insiders  
❌ Team can unlock trước investors  

---

## PHẦN 3: REWARD SYSTEMS

### 8. Staking & LP Rewards

#### **8.1. Staking Mechanics**

**Basic staking**:
```
User deposits MBX → Receives sMBX (staked token)
sMBX accrues rewards over time (fees + emissions)
User can unstake anytime (or with timelock)
```

**Tiered staking** (Recommended cho perp DEX):
```
No lock: 8-15% APY, 0% discount
3 months: 15-25% APY, 5% discount
6 months: 25-40% APY, 10% discount
12 months: 40-70% APY, 15% discount
```

**Implementation note**:
```solidity
struct StakeTier {
    uint256 minDuration; // 0, 90, 180, 365 days
    uint256 multiplier;  // 100, 150, 250, 400 (basis points)
    uint256 discount;    // 0, 500, 1000, 1500 (0-15%)
}

mapping(address => StakeInfo) public stakes;

struct StakeInfo {
    uint256 amount;
    uint256 startTime;
    uint256 lockDuration;
    uint256 tier; // 0-3
}
```

#### **8.2. LP Rewards**

**Liquidity mining**:
```
User provides MBX/USDC liquidity → Receives LP tokens
LP tokens staked → Earn MBX emissions (bonus APR)
Duration: 6-12 tháng, taper dần
```

**Concentrated liquidity** (Uniswap V3 style):
```
User provides liquidity in specific range
Tighter range = higher APR multiplier
Example: $0.45-$0.55 range → 2x rewards
```

#### **8.3. Reward Calculation**

**RewardPerToken pattern** (gas-efficient O(1)):
```solidity
uint256 public rewardPerTokenStored;
mapping(address => uint256) public userRewardPerTokenPaid;
mapping(address => uint256) public rewards;

function rewardPerToken() public view returns (uint256) {
    if (totalSupply == 0) return rewardPerTokenStored;
    
    return rewardPerTokenStored + (
        (block.timestamp - lastUpdateTime) * rewardRate * 1e18 / totalSupply
    );
}

function earned(address account) public view returns (uint256) {
    return (
        balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18
    ) + rewards[account];
}

modifier updateReward(address account) {
    rewardPerTokenStored = rewardPerToken();
    lastUpdateTime = block.timestamp;
    rewards[account] = earned(account);
    userRewardPerTokenPaid[account] = rewardPerTokenStored;
    _;
}
```

---

### 9. Emissions Schedule & Taper

**Emissions = Mint new tokens làm reward**, phải taper để avoid inflation.

#### **9.1. Standard Emissions Curve**

```
Bootstrap Phase (0-3 months): 100% emission rate
├─ Target: 30-40% APY để attract early adopters
├─ Risk: High inflation, cần balance với volume

Growth Phase (3-12 months): Taper 50%
├─ Emissions giảm dần theo volume tăng
├─ Target: 20-30% APY

Maturity Phase (12+ months): Taper 20%
├─ Real yield từ fees > emissions
├─ Target: 10-20% APY (sustainable)
```

#### **9.2. Dynamic Emissions**

**Target staking ratio** (Recommended):
```
Goal: 40-60% supply staked

If staked < 40%: Increase emission rate → attract more stakers
If staked > 60%: Decrease emission rate → reduce sell pressure

Formula:
emissionMultiplier = targetRatio / actualRatio
adjustedRate = baseRate * emissionMultiplier
```

**Example**:
```
Base rate: 100 MBX/day
Target: 50% staked
Actual: 30% staked

Multiplier = 50 / 30 = 1.67
Adjusted rate = 100 * 1.67 = 167 MBX/day (boost rewards)
```

#### **9.3. Emissions Cap**

```solidity
uint256 public constant EMISSION_CAP = 20_000_000 * 1e18; // 20M MBX
uint256 public emissionStartTime;
uint256 public emissionDuration = 365 days;
uint256 public totalEmitted;

function currentEmissionRate() public view returns (uint256) {
    if (block.timestamp > emissionStartTime + emissionDuration) {
        return 0; // Emissions ended
    }
    
    uint256 elapsed = block.timestamp - emissionStartTime;
    uint256 remaining = EMISSION_CAP - totalEmitted;
    uint256 timeLeft = emissionDuration - elapsed;
    
    // Linear taper
    return remaining / timeLeft;
}
```

---

### 10. Fee Distribution Models

**Protocol fees = Revenue stream**, chia cho stakeholders.

#### **10.1. Fee Types Trong Perp DEX**

```
Trading Fees:
├─ Open position: 0.06-0.10%
├─ Close position: 0.06-0.10%
├─ Price impact: Variable
└─ Funding rate: Hourly (long vs short balance)

Borrowing Fees:
└─ Borrow from pool: 0.01%/hour

Liquidation Fees:
└─ 1-5% of liquidated position
```

#### **10.2. Distribution Models**

**GMX Model** (Balanced):
```
Total fees: 100%
├─ Liquidity Providers (GLP holders): 63%
├─ GMX stakers: 27%
└─ Treasury: 10%
```

**Gains Network** (Burn-Heavy):
```
Total fees: 100%
├─ Liquidity (DAI vault): 45%
└─ Buyback & burn GNS: 55%
```

**Hyperliquid** (Token-Holder Focus):
```
Total fees: 100%
├─ Liquidity: Minimal (native orderbook)
└─ HYPE stakers: 97-99%
```

**MegaBanX Proposal** (Recommended):
```
Total fees: 100%
├─ USDm Liquidity Providers: 60%
├─ MBX stakers: 30%
└─ Buyback & burn + treasury: 10%
   ├─ Burn: 7%
   └─ Treasury buffer: 3%
```

#### **10.3. Implementation Pattern**

```solidity
contract FeeRouter {
    address public usdmPool;
    address public stakingContract;
    address public treasury;
    address public buybackContract;
    
    uint256 public constant LP_SHARE = 6000; // 60%
    uint256 public constant STAKER_SHARE = 3000; // 30%
    uint256 public constant BURN_SHARE = 700; // 7%
    uint256 public constant TREASURY_SHARE = 300; // 3%
    
    function distributeFees(uint256 totalFees) external {
        uint256 lpAmount = totalFees * LP_SHARE / 10000;
        uint256 stakerAmount = totalFees * STAKER_SHARE / 10000;
        uint256 burnAmount = totalFees * BURN_SHARE / 10000;
        uint256 treasuryAmount = totalFees * TREASURY_SHARE / 10000;
        
        USDC.transfer(usdmPool, lpAmount);
        USDC.transfer(stakingContract, stakerAmount);
        USDC.transfer(buybackContract, burnAmount);
        USDC.transfer(treasury, treasuryAmount);
        
        emit FeesDistributed(lpAmount, stakerAmount, burnAmount, treasuryAmount);
    }
}
```

---

### 11. Buyback & Burn Mechanism

**Buyback & burn = Dùng protocol revenue mua token rồi đốt**, tạo deflationary pressure.

#### **11.1. Flow Chart**

```
Weekly Buyback Cycle:
1. Accumulate fees (USDC) in BuybackContract
2. Bot/keeper trigger buyback() function
3. Swap USDC → MBX on DEX (best price)
4. Transfer MBX to 0x000...dead address
5. Emit Burn event (on-chain proof)
```

#### **11.2. Buyback Strategies**

**Time-Weighted Average (TWAP)** (Recommended):
```
Instead of 1 big buy weekly:
→ Buy small amounts every hour (168 buys/week)
→ Less price impact, more stable

Example:
Weekly budget: 7000 USDC
Hourly buy: 7000 / 168 = 41.67 USDC/hour
```

**Dutch Auction**:
```
Start price: Market price + 5%
End price: Market price - 5%
Duration: 7 days
→ Buy at best available price in range
```

**Conditional Buyback**:
```
Only buyback when:
- Price < 7-day MA (support dips)
- Volume > threshold (avoid manipulation)
- Treasury reserve > X (safety buffer)
```

#### **11.3. Burn Implementation**

```solidity
contract BuybackBurn {
    IERC20 public MBX;
    IERC20 public USDC;
    IUniswapV2Router public router;
    
    address public constant DEAD = 0x000000000000000000000000000000000000dEaD;
    
    uint256 public totalBurned;
    uint256 public totalUSDCSpent;
    
    event Buyback(uint256 usdcAmount, uint256 mbxReceived);
    event Burn(uint256 mbxAmount, uint256 timestamp);
    
    function executeBuyback(uint256 usdcAmount) external onlyKeeper {
        // 1. Approve router
        USDC.approve(address(router), usdcAmount);
        
        // 2. Swap USDC → MBX
        address[] memory path = new address[](2);
        path[0] = address(USDC);
        path[1] = address(MBX);
        
        uint256[] memory amounts = router.swapExactTokensForTokens(
            usdcAmount,
            0, // Accept any amount (add slippage in production)
            path,
            address(this),
            block.timestamp + 300
        );
        
        uint256 mbxReceived = amounts[1];
        emit Buyback(usdcAmount, mbxReceived);
        
        // 3. Burn MBX
        MBX.transfer(DEAD, mbxReceived);
        totalBurned += mbxReceived;
        totalUSDCSpent += usdcAmount;
        
        emit Burn(mbxReceived, block.timestamp);
    }
    
    function burnRate() external view returns (uint256) {
        // Annual burn rate as % of supply
        // Example: 2M burned / 100M supply = 2% annual
        return (totalBurned * 10000) / MBX.totalSupply();
    }
}
```

#### **11.4. Expected Impact**

```
Scenario: MegaBanX @ $500M TVL

Assumptions:
- Daily volume: $50M
- Avg fee: 0.07% = $35k/day fees
- Burn allocation: 7% = $2,450/day
- MBX price: $0.50

Calculations:
Daily burn: $2,450 / $0.50 = 4,900 MBX
Annual burn: 4,900 * 365 = 1,788,500 MBX
Burn rate: 1.79M / 100M = 1.79%/year

At $2B TVL (4x volume):
→ Annual burn: 7.16M MBX = 7.16%/year
```

---

## PHẦN 4: GOVERNANCE & VALUE ACCRUAL

### 12. Governance Models

#### **12.1. Basic (1 Token = 1 Vote)**

```solidity
contract BasicGovernance {
    mapping(uint256 => Proposal) public proposals;
    
    struct Proposal {
        string description;
        uint256 forVotes;
        uint256 againstVotes;
        uint256 deadline;
        bool executed;
    }
    
    function vote(uint256 proposalId, bool support) external {
        uint256 votes = MBX.balanceOf(msg.sender);
        require(votes > 0, "No voting power");
        
        if (support) {
            proposals[proposalId].forVotes += votes;
        } else {
            proposals[proposalId].againstVotes += votes;
        }
    }
}
```

**Pros**: Simple, transparent  
**Cons**: Whales control votes

#### **12.2. veToken Model (Curve-Style)**

**Concept**: Lock token lâu hơn = voting power nhiều hơn.

```
Lock 100 MBX:
- 3 months → 25 veMBX
- 6 months → 50 veMBX
- 1 year → 100 veMBX
- 4 years → 400 veMBX (max 4x boost)

veMBX decays linear về 0 khi lock expires.
```

**Implementation**:
```solidity
contract VotingEscrow {
    struct LockedBalance {
        uint256 amount;
        uint256 end; // Unlock timestamp
    }
    
    mapping(address => LockedBalance) public locked;
    
    function balanceOf(address user) public view returns (uint256) {
        LockedBalance memory _locked = locked[user];
        if (_locked.end <= block.timestamp) return 0;
        
        // Linear decay
        uint256 timeLeft = _locked.end - block.timestamp;
        uint256 maxTime = 4 * 365 days;
        
        return (_locked.amount * timeLeft) / maxTime;
    }
    
    function createLock(uint256 amount, uint256 duration) external {
        require(duration >= 7 days && duration <= 4 * 365 days);
        
        MBX.transferFrom(msg.sender, address(this), amount);
        locked[msg.sender] = LockedBalance({
            amount: amount,
            end: block.timestamp + duration
        });
    }
}
```

**Pros**: Long-term alignment, reduce whale power  
**Cons**: Complex, UX friction

#### **12.3. Quadratic Voting**

**Formula**: Cost = votes²

```
1 vote costs 1 token
2 votes cost 4 tokens
3 votes cost 9 tokens
10 votes cost 100 tokens
```

**Pros**: Fair distribution, reduce plutocracy  
**Cons**: Gas-heavy, Sybil attack risk

---

### 13. Value Accrual

**Làm sao token tăng giá theo time?**

#### **13.1. Value Accrual Mechanisms**

**Direct Revenue Share**:
```
Protocol earnings → Distributed to stakers (USDm/USDC)
Example: GMX, MegaBanX
→ Real yield model
```

**Buyback & Burn**:
```
Protocol earnings → Buy token → Burn
Example: BNB, Gains Network
→ Supply reduction → Price up
```

**veToken Boost**:
```
Lock token → Earn higher APY on LP/staking
Example: Curve (veCRV boost)
→ Lock demand → Circulating supply down
```

**Governance Power**:
```
Hold token → Control treasury, fees, incentives
Example: MakerDAO (MKR), Compound (COMP)
→ Valuable if protocol big
```

**Fee Discount**:
```
Stake token → Trade cheaper
Example: BNB (Binance 25% discount), MBX (15% discount)
→ Usage demand
```

#### **13.2. Combining Mechanisms (Best Practice)**

```
MegaBanX MBX Value Accrual:

1. Revenue Share (30% fees → stakers) ← Real yield
2. Buyback & Burn (7% fees) ← Deflationary
3. Fee Discount (0-15%) ← Usage demand
4. Governance (DAO votes) ← Control value
5. veToken boost (multipliers) ← Lock demand

Result:
- Short-term: Fee discount + revenue → usage demand
- Long-term: Burn + lock → supply shock → price up
```

---

## PHẦN 5: TRIỂN KHAI SMART CONTRACT

### 14. Contract Architecture Overview

#### **14.1. System Diagram**

```
┌─────────────────────────────────────────────────┐
│              MegaBanX Protocol                  │
│  (Perp Trading + Liquidity + Staking System)   │
└─────────────────────────────────────────────────┘
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
┌─────────────┐  ┌──────────────┐  ┌──────────────┐
│  MBX Token  │  │  esMBX Token │  │  USDm Pool   │
│  (ERC20)    │  │(Non-Transfer)│  │  (Liquidity) │
└─────────────┘  └──────────────┘  └──────────────┘
        │               │                │
        └───────────────┼────────────────┘
                        │
                        ▼
        ┌───────────────────────────────┐
        │       Fee Router              │
        │  (Collect & Distribute Fees)  │
        └───────────────────────────────┘
                │       │       │
        ┌───────┘       │       └───────┐
        │               │               │
        ▼               ▼               ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Staking    │  │   Buyback    │  │   Treasury   │
│  (MBX Vault) │  │  & Burn      │  │   (MultiSig) │
└──────────────┘  └──────────────┘  └──────────────┘
        │               │
        ▼               ▼
┌──────────────┐  ┌──────────────┐
│   Vesting    │  │  Governance  │
│  (esM→MBX)   │  │  (veTokens)  │
└──────────────┘  └──────────────┘
```

#### **14.2. Contract Checklist**

```
Core Contracts:
✅ MBX.sol (ERC20 token)
✅ esMBX.sol (Non-transferable escrowed token)
✅ StakingRewards.sol (Stake MBX earn rewards)
✅ VestingVault.sol (esMBX → MBX vesting)
✅ FeeRouter.sol (Distribute protocol fees)
✅ BuybackBurn.sol (Weekly buyback & burn)
✅ Treasury.sol (Gnosis Safe multisig)
✅ Governance.sol (Voting, proposals)

Supporting:
✅ RewardDistributor.sol (Emissions logic)
✅ TierManager.sol (Lock duration multipliers)
✅ FeeCollector.sol (Aggregate perp fees)
```

---

### 15. ERC20 + Non-Transferable esToken

#### **15.1. MBX Token (Standard ERC20)**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MBXToken is ERC20, Ownable {
    uint256 public constant MAX_SUPPLY = 100_000_000 * 1e18;
    
    constructor() ERC20("MegaBanX", "MBX") Ownable(msg.sender) {
        // Mint to deployer (then distribute to contracts)
        _mint(msg.sender, MAX_SUPPLY);
    }
    
    // Optional: Renounce ownership after setup
    function renounceOwnership() public override onlyOwner {
        super.renounceOwnership();
    }
}
```

#### **15.2. esMBX Token (Non-Transferable)**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

contract esMBXToken is ERC20, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    
    constructor() ERC20("Escrowed MBX", "esMBX") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
    
    // Only StakingRewards can mint
    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        _mint(to, amount);
    }
    
    // Only VestingVault can burn
    function burn(address from, uint256 amount) external onlyRole(MINTER_ROLE) {
        _burn(from, amount);
    }
    
    // CRITICAL: Disable transfers
    function transfer(address, uint256) public pure override returns (bool) {
        revert("esMBX: non-transferable");
    }
    
    function transferFrom(address, address, uint256) public pure override returns (bool) {
        revert("esMBX: non-transferable");
    }
    
    // Allow approve for VestingVault to burn
    function approve(address spender, uint256 amount) public virtual override returns (bool) {
        if (hasRole(MINTER_ROLE, spender)) {
            return super.approve(spender, amount);
        }
        revert("esMBX: approve only for vesting");
    }
}
```

---

### 16. Staking Rewards Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

interface IesMBX {
    function mint(address to, uint256 amount) external;
}

contract StakingRewards is ReentrancyGuard, Ownable {
    IERC20 public immutable stakingToken; // MBX
    IERC20 public immutable rewardToken;  // USDC (from fees)
    IesMBX public immutable esMBX;
    
    // Tier structure
    struct Tier {
        uint256 minDuration;
        uint256 multiplier; // Basis points (10000 = 1x)
        uint256 discount;   // Basis points (1500 = 15%)
    }
    
    Tier[4] public tiers;
    
    // Staker info
    struct StakeInfo {
        uint256 amount;
        uint256 startTime;
        uint256 lockEnd;
        uint8 tier;
    }
    
    mapping(address => StakeInfo) public stakes;
    
    // Rewards accounting (O(1) pattern)
    uint256 public rewardPerTokenStored;
    uint256 public lastUpdateTime;
    uint256 public totalStaked;
    
    // esMBX emissions
    uint256 public emissionRate = 100 * 1e18; // 100 esMBX/second
    uint256 public emissionEnd;
    
    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards;
    
    event Staked(address indexed user, uint256 amount, uint8 tier);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardPaid(address indexed user, uint256 usdcReward, uint256 esMBXReward);
    
    constructor(
        address _stakingToken,
        address _rewardToken,
        address _esMBX
    ) Ownable(msg.sender) {
        stakingToken = IERC20(_stakingToken);
        rewardToken = IERC20(_rewardToken);
        esMBX = IesMBX(_esMBX);
        
        // Initialize tiers
        tiers[0] = Tier(0, 10000, 0);          // No lock: 1x, 0%
        tiers[1] = Tier(90 days, 15000, 500);  // 3mo: 1.5x, 5%
        tiers[2] = Tier(180 days, 25000, 1000); // 6mo: 2.5x, 10%
        tiers[3] = Tier(365 days, 40000, 1500); // 12mo: 4x, 15%
        
        emissionEnd = block.timestamp + 365 days;
    }
    
    // Reward calculation
    function rewardPerToken() public view returns (uint256) {
        if (totalStaked == 0) return rewardPerTokenStored;
        
        uint256 timeElapsed = block.timestamp - lastUpdateTime;
        uint256 newRewards = timeElapsed * emissionRate;
        
        return rewardPerTokenStored + (newRewards * 1e18 / totalStaked);
    }
    
    function earned(address account) public view returns (uint256, uint256) {
        StakeInfo memory stake = stakes[account];
        if (stake.amount == 0) return (0, 0);
        
        // Apply tier multiplier
        uint256 effectiveBalance = (stake.amount * tiers[stake.tier].multiplier) / 10000;
        
        // USDC rewards from fees
        uint256 usdcReward = (
            effectiveBalance * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18
        ) + rewards[account];
        
        // esMBX emissions
        uint256 esMBXReward = (
            effectiveBalance * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18
        );
        
        return (usdcReward, esMBXReward);
    }
    
    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;
        
        if (account != address(0)) {
            (uint256 usdcReward, uint256 esMBXReward) = earned(account);
            rewards[account] = usdcReward;
        }
        _;
    }
    
    // Stake with tier
    function stake(uint256 amount, uint8 tier) external nonReentrant updateReward(msg.sender) {
        require(amount > 0, "Cannot stake 0");
        require(tier < 4, "Invalid tier");
        require(stakes[msg.sender].amount == 0, "Already staked, withdraw first");
        
        stakingToken.transferFrom(msg.sender, address(this), amount);
        
        stakes[msg.sender] = StakeInfo({
            amount: amount,
            startTime: block.timestamp,
            lockEnd: block.timestamp + tiers[tier].minDuration,
            tier: tier
        });
        
        totalStaked += amount;
        emit Staked(msg.sender, amount, tier);
    }
    
    // Withdraw (after lock)
    function withdraw() external nonReentrant updateReward(msg.sender) {
        StakeInfo memory stakeInfo = stakes[msg.sender];
        require(stakeInfo.amount > 0, "No stake");
        require(block.timestamp >= stakeInfo.lockEnd, "Still locked");
        
        uint256 amount = stakeInfo.amount;
        totalStaked -= amount;
        delete stakes[msg.sender];
        
        stakingToken.transfer(msg.sender, amount);
        emit Withdrawn(msg.sender, amount);
    }
    
    // Claim rewards
    function claimRewards() external nonReentrant updateReward(msg.sender) {
        (uint256 usdcReward, uint256 esMBXReward) = earned(msg.sender);
        
        if (usdcReward > 0) {
            rewards[msg.sender] = 0;
            rewardToken.transfer(msg.sender, usdcReward);
        }
        
        if (esMBXReward > 0 && block.timestamp < emissionEnd) {
            esMBX.mint(msg.sender, esMBXReward);
        }
        
        emit RewardPaid(msg.sender, usdcReward, esMBXReward);
    }
    
    // Get user's trading discount
    function getUserDiscount(address user) external view returns (uint256) {
        return tiers[stakes[user].tier].discount;
    }
    
    // Owner: Distribute USDC fees
    function notifyRewardAmount(uint256 amount) external onlyOwner updateReward(address(0)) {
        rewardToken.transferFrom(msg.sender, address(this), amount);
    }
}
```

---

### 17. Vesting Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

interface IesMBX {
    function burn(address from, uint256 amount) external;
}

contract VestingVault is ReentrancyGuard {
    IERC20 public immutable MBX;
    IesMBX public immutable esMBX;
    
    uint256 public constant VESTING_DURATION = 365 days;
    
    struct VestingSchedule {
        uint256 totalAmount;
        uint256 startTime;
        uint256 claimed;
    }
    
    mapping(address => VestingSchedule) public schedules;
    
    event VestingStarted(address indexed user, uint256 amount);
    event Claimed(address indexed user, uint256 amount);
    
    constructor(address _mbx, address _esMBX) {
        MBX = IERC20(_mbx);
        esMBX = IesMBX(_esMBX);
    }
    
    // Start vesting esMBX → MBX
    function startVesting(uint256 amount) external nonReentrant {
        require(amount > 0, "Amount = 0");
        require(schedules[msg.sender].totalAmount == 0, "Already vesting");
        
        // Burn esMBX (user must approve first)
        esMBX.burn(msg.sender, amount);
        
        schedules[msg.sender] = VestingSchedule({
            totalAmount: amount,
            startTime: block.timestamp,
            claimed: 0
        });
        
        emit VestingStarted(msg.sender, amount);
    }
    
    // Calculate claimable MBX
    function claimable(address user) public view returns (uint256) {
        VestingSchedule memory schedule = schedules[user];
        if (schedule.totalAmount == 0) return 0;
        
        uint256 elapsed = block.timestamp - schedule.startTime;
        
        if (elapsed >= VESTING_DURATION) {
            return schedule.totalAmount - schedule.claimed;
        }
        
        uint256 vested = (schedule.totalAmount * elapsed) / VESTING_DURATION;
        return vested - schedule.claimed;
    }
    
    // Claim vested MBX
    function claim() external nonReentrant {
        uint256 amount = claimable(msg.sender);
        require(amount > 0, "Nothing to claim");
        
        schedules[msg.sender].claimed += amount;
        MBX.transfer(msg.sender, amount);
        
        emit Claimed(msg.sender, amount);
    }
}
```

---

### 18. Fee Router & Buyback Contract

#### **18.1. Fee Router**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract FeeRouter is Ownable {
    IERC20 public immutable USDC;
    
    address public usdmPool;
    address public stakingContract;
    address public buybackContract;
    address public treasury;
    
    uint256 public constant LP_SHARE = 6000;      // 60%
    uint256 public constant STAKER_SHARE = 3000;  // 30%
    uint256 public constant BURN_SHARE = 700;     // 7%
    uint256 public constant TREASURY_SHARE = 300; // 3%
    
    event FeesDistributed(
        uint256 lpAmount,
        uint256 stakerAmount,
        uint256 burnAmount,
        uint256 treasuryAmount
    );
    
    constructor(
        address _usdc,
        address _usdmPool,
        address _stakingContract,
        address _buybackContract,
        address _treasury
    ) Ownable(msg.sender) {
        USDC = IERC20(_usdc);
        usdmPool = _usdmPool;
        stakingContract = _stakingContract;
        buybackContract = _buybackContract;
        treasury = _treasury;
    }
    
    function distributeFees(uint256 totalFees) external {
        require(USDC.balanceOf(address(this)) >= totalFees, "Insufficient balance");
        
        uint256 lpAmount = (totalFees * LP_SHARE) / 10000;
        uint256 stakerAmount = (totalFees * STAKER_SHARE) / 10000;
        uint256 burnAmount = (totalFees * BURN_SHARE) / 10000;
        uint256 treasuryAmount = (totalFees * TREASURY_SHARE) / 10000;
        
        USDC.transfer(usdmPool, lpAmount);
        USDC.transfer(stakingContract, stakerAmount);
        USDC.transfer(buybackContract, burnAmount);
        USDC.transfer(treasury, treasuryAmount);
        
        emit FeesDistributed(lpAmount, stakerAmount, burnAmount, treasuryAmount);
    }
}
```

#### **18.2. Buyback & Burn**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}

contract BuybackBurn is Ownable {
    IERC20 public immutable MBX;
    IERC20 public immutable USDC;
    IUniswapV2Router public immutable router;
    
    address public constant DEAD = 0x000000000000000000000000000000000000dEaD;
    
    uint256 public totalBurned;
    uint256 public totalUSDCSpent;
    uint256 public lastBuybackTime;
    
    event Buyback(uint256 usdcAmount, uint256 mbxReceived, uint256 timestamp);
    event Burn(uint256 mbxAmount, uint256 timestamp);
    
    constructor(
        address _mbx,
        address _usdc,
        address _router
    ) Ownable(msg.sender) {
        MBX = IERC20(_mbx);
        USDC = IERC20(_usdc);
        router = IUniswapV2Router(_router);
    }
    
    // Execute weekly buyback
    function executeBuyback() external onlyOwner {
        uint256 usdcBalance = USDC.balanceOf(address(this));
        require(usdcBalance > 0, "No USDC to buyback");
        require(block.timestamp >= lastBuybackTime + 7 days, "Too soon");
        
        // Approve router
        USDC.approve(address(router), usdcBalance);
        
        // Swap path
        address[] memory path = new address[](2);
        path[0] = address(USDC);
        path[1] = address(MBX);
        
        // Execute swap
        uint256[] memory amounts = router.swapExactTokensForTokens(
            usdcBalance,
            0, // TODO: Add slippage protection in production
            path,
            address(this),
            block.timestamp + 300
        );
        
        uint256 mbxReceived = amounts[1];
        totalUSDCSpent += usdcBalance;
        lastBuybackTime = block.timestamp;
        
        emit Buyback(usdcBalance, mbxReceived, block.timestamp);
        
        // Burn MBX
        MBX.transfer(DEAD, mbxReceived);
        totalBurned += mbxReceived;
        
        emit Burn(mbxReceived, block.timestamp);
    }
    
    // View annual burn rate
    function annualBurnRate() external view returns (uint256) {
        if (totalBurned == 0) return 0;
        
        // Burn as % of total supply
        uint256 totalSupply = MBX.totalSupply();
        return (totalBurned * 10000) / totalSupply; // Basis points
    }
}
```

---

## PHẦN 6: LAUNCH STRATEGY & LONG-TERM PLAN

### 19. Bootstrap Liquidity

#### **19.1. Pre-Launch Checklist (T-4 weeks)**

```
Week -4 to -3: Smart Contracts
✅ Deploy all contracts on testnet
✅ Integration testing
✅ Security audit (CertiK/Trail of Bits)
✅ Fix critical issues
✅ Deploy on mainnet (MegaETH)
✅ Verify contracts on explorer

Week -3 to -2: Token Distribution
✅ Mint 100M MBX to deployer
✅ Transfer allocations:
   - Treasury: 25M MBX
   - Staking rewards: 20M MBX
   - LP incentives: 10M MBX
   - DEX liquidity: 5M MBX
   - Team vesting contract: 20M MBX
   - Investor vesting: 15M MBX
✅ Renounce MBX minting (make immutable)

Week -2 to -1: Liquidity Setup
✅ Add initial liquidity MBX/USDC (5M MBX + $250K USDC)
✅ Lock LP tokens (Unicrypt/Team Finance)
✅ Activate staking contract
✅ Configure fee router
✅ Set emission rates

Week -1 to Launch: Marketing
✅ Announce tokenomics publicly
✅ Publish docs (GitBook)
✅ Twitter/Discord campaigns
✅ KOL partnerships
✅ Testnet trading competition
```

#### **19.2. Initial Liquidity Strategy**

**Option A: Concentrated Liquidity (Uniswap V3)**
```
Range: $0.40 - $0.60 (tight)
Capital: 5M MBX + $250K USDC
Expected TVL: $500K (assuming $0.50 price)
Pros: Capital efficient, high APR for LPs
Cons: Need to rebalance if price moves
```

**Option B: Full-Range (Uniswap V2)**
```
Classic xy=k pool
Capital: 5M MBX + $250K USDC
Expected TVL: $500K
Pros: No rebalance needed, simple
Cons: Less capital efficient
```

**Recommended: Hybrid**
```
80% Concentrated ($0.45-$0.55)
20% Full-range (backup)
```

#### **19.3. Bootstrapping Incentives**

**First 30 Days Plan**:
```
Objective: Reach $5M TVL staked + $50M daily perp volume

Tactics:
1. High APY staking (40-70% via emissions)
2. Trading contests ($50K MBX prizes)
3. Referral bonuses (20% extra for early users)
4. LP mining (2x rewards first month)
5. Airdrops to testnet users (500K MBX total)

Budget:
- Emissions: 2M MBX (20% of pool)
- Prizes: 500K MBX
- Marketing: $100K (from treasury)
```

---

### 20. Go-to-Market Plan

#### **20.1. Phase 1: Launch (Month 1-3)**

**Goals**:
- 1,000+ traders
- $10M+ daily volume
- $2M+ TVL staked
- 40%+ staking ratio

**Tactics**:
```
Week 1-2: Early Adopter Phase
- Whitelist 100 beta testers
- 2x rewards for first 1000 stakers
- Bug bounty program ($50K pool)
- Daily Twitter spaces

Week 3-6: Volume Ramp
- Trading competitions (top PNL wins MBX)
- Referral program launch (10-20% rebates)
- KOL partnerships (3-5 tier-1 influencers)
- Cross-promo with MegaETH ecosystem

Week 7-12: Optimization
- Fee structure tweaks based on data
- Add new perp markets (user voting)
- Emissions taper 25% (maintain APY via volume)
- Launch DAO governance alpha
```

#### **20.2. Phase 2: Growth (Month 4-9)**

**Goals**:
- 10,000+ traders
- $100M+ daily volume
- $20M+ TVL
- Real yield > emissions

**Tactics**:
```
- Institutional onboarding (MM, whales)
- Integration với aggregators (1inch, CoW)
- Cross-chain bridge (if applicable)
- Advanced features (copy trading, algo orders)
- Partnership với other protocols
- Conference sponsorships
```

#### **20.3. Phase 3: Maturity (Month 10+)**

**Goals**:
- Sustainable APY from fees (15-25%)
- Emissions <20% of staking rewards
- Top 5 perp DEX by volume
- Decentralized governance

**Tactics**:
```
- Launch veToken (veMBX) for governance
- Revenue diversification (perp + spot + options)
- Buyback velocity increase
- Treasury yield strategies
- Ecosystem grants program
```

---

### 21. Metrics Tracking & KPI

#### **21.1. Core Metrics Dashboard**

**Daily Tracking**:
```
Protocol Metrics:
├─ Daily volume (USD)
├─ Open interest (USD)
├─ Number of trades
├─ Unique traders (24h)
├─ Liquidations (count + value)
└─ Fees generated (USD)

Token Metrics:
├─ MBX price (USD)
├─ Market cap
├─ Fully diluted valuation (FDV)
├─ Circulating supply
├─ Trading volume (MBX/USDC pair)
└─ Holder count

Staking Metrics:
├─ Total staked (MBX)
├─ Staking ratio (%)
├─ APY (real-time)
├─ Tier distribution (no lock, 3mo, 6mo, 12mo)
└─ esMBX minted vs vested

TVL Metrics:
├─ USDm pool size
├─ LP APY
├─ Utilization rate
└─ PnL (cumulative trader vs pool)
```

#### **21.2. KPI Targets**

**Month 1**:
```
Daily Volume: $10M
TVL: $2M
Stakers: 500
APY: 40-50% (bootstrap)
```

**Month 3**:
```
Daily Volume: $50M
TVL: $10M
Stakers: 2,000
APY: 30-40%
```

**Month 6**:
```
Daily Volume: $150M
TVL: $50M
Stakers: 10,000
APY: 20-30% (fees > 50% of rewards)
```

**Month 12**:
```
Daily Volume: $500M
TVL: $200M
Stakers: 50,000
APY: 15-25% (fees > 80% of rewards)
```

#### **21.3. Tools**

```
On-Chain Analytics:
- Dune Analytics (custom dashboard)
- The Graph (subgraph)
- Nansen (holder tracking)

Off-Chain:
- Google Analytics (website traffic)
- Discord/Twitter metrics (community growth)
- CoinGecko/CMC listings

Internal:
- Grafana (real-time monitoring)
- Custom admin panel (Next.js)
- Alerts (Telegram bot for critical events)
```

---

### 22. Common Pitfalls & How to Avoid

#### **22.1. Tokenomics Death Spirals**

**Problem**: High emissions → Dump → Price crash → APY crash → Exit spiral.

**Solution**:
```
✅ Cap emissions (20-30% supply max)
✅ Taper aggressively (50% reduction Month 3)
✅ Vesting (esMBX model)
✅ Dynamic rates (adjust based on staking ratio)
✅ Buyback pressure (burn supports floor)
```

**Example**:
```
Month 1: 40% APY (50% emissions + 50% fees)
Month 3: 30% APY (30% emissions + 70% fees)
Month 6: 20% APY (10% emissions + 90% fees)
Month 12: 15% APY (0% emissions + 100% fees)
```

---

#### **22.2. Governance Attacks**

**Problem**: Whale buys governance token → Votes malicious proposal → Rug treasury.

**Solution**:
```
✅ Timelock (48-72h delay on execution)
✅ Multisig veto (3/5 team can cancel)
✅ veToken (lock requirement for votes)
✅ Quorum (minimum 10% supply must vote)
✅ Gradual decentralization (DAO Year 2+)
```

---

#### **22.3. Liquidity Crises**

**Problem**: Pool imbalanced → High slippage → Users leave → Death spiral.

**Solution**:
```
✅ Dynamic fees (increase when imbalanced)
✅ Deposit bonuses (incentivize rebalancing)
✅ Backup liquidity (treasury AMM)
✅ Circuit breakers (pause if utilization >95%)
✅ Insurance fund (cover liquidation gaps)
```

---

#### **22.4. Regulatory Risk**

**Problem**: Token classified as security → Legal trouble.

**Solution**:
```
✅ Fair launch (no pre-mine to team)
✅ Decentralized from start (no company ownership)
✅ Utility-first (governance + fee discount, not investment)
✅ Legal opinion (consult crypto lawyer)
✅ Geo-blocking (restrict US if needed)
```

---

## PHẦN 7: CASE STUDIES

### 23. GMX Tokenomics Deep Dive

#### **23.1. GMX Token Mechanics**

```
Token: GMX (fixed supply: ~10M)
Utility:
- Stake → Earn 27% protocol fees (ETH/AVAX)
- Governance votes
- No fee discount (different from MBX)

Rewards:
- esGMX (vesting 365 days → GMX)
- Multiplier Points (boost APR up to 2x)
- ETH/AVAX fees (real yield)

Results (2023-2024):
- APY: 20-36% sustained
- Staking ratio: 80%+ (very high)
- Fee revenue: $200M+ cumulative
- Token price: $30-80 range (survived bear)
```

#### **23.2. Key Learnings**

```
✅ esGMX vesting worked → Low sell pressure
✅ Real yield attractive → Institutional interest
✅ Multiplier points gamified → High retention
✅ No buyback needed → Fees alone sufficient
✅ GLP (liquidity) decoupled from GMX → Composability

❌ esGMX complex UX → Confusion for new users
❌ No fee discount → Less trading utility
❌ High concentration → Whale control
```

**Applied to MegaBanX**:
```
Copy: esMBX vesting, real yield, multipliers
Improve: Add fee discount, more aggressive taper
```

---

### 24. Curve veToken Model

#### **24.1. veCRV Mechanics**

```
Token: CRV (inflationary, decreasing emissions)
Utility:
- Lock CRV (up to 4 years) → veCRV
- veCRV boost LP APR (up to 2.5x)
- Governance votes (gauge weights)
- 50% trading fees

Lock Formula:
100 CRV locked 4 years = 100 veCRV (max)
100 CRV locked 1 year = 25 veCRV
veCRV decays linearly to 0 at unlock

Results:
- 40%+ CRV locked perpetually
- Bribe markets emerged (Convex, Votium)
- Governance power commoditized
```

#### **24.2. Key Learnings**

```
✅ veToken creates sticky liquidity
✅ Long-term alignment (4-year lock)
✅ Governance valuable (gauge votes = revenue)
✅ Composability (Convex built on top)

❌ Complex UX (lock, boost, vote)
❌ Inflationary (emissions too high early)
❌ Bribe meta centralized power
```

**Applied to MegaBanX**:
```
Phase 2: Introduce veMBX for governance
- Lock MBX → veMBX (boost staking APR)
- Vote on fee splits, new markets
- Simpler than Curve (no gauge complexity)
```

---

### 25. MegaBanX Example Implementation

#### **25.1. Full Tokenomics Summary**

```
═══════════════════════════════════════════════════
         MEGABANX TOKENOMICS FINAL SPEC
═══════════════════════════════════════════════════

TOKEN: MBX
├─ Total Supply: 100,000,000 (fixed)
├─ Chain: MegaETH
├─ Standard: ERC20
└─ Bridge: 1:1 from ABX (Alephium)

ALLOCATION:
├─ Team/Advisors: 20M (20%) - 4yr vest, 1yr cliff
├─ Investors: 15M (15%) - 2yr vest, 6mo cliff
├─ Treasury/DAO: 25M (25%) - Governance control
├─ Staking Rewards: 20M (20%) - 12mo emissions
├─ LP Incentives: 10M (10%) - 6mo emissions
├─ Public Sale: 5M (5%) - No vesting
└─ DEX Liquidity: 5M (5%) - Locked

UTILITY:
1. Fee Discount (0-15% based on lock)
2. Revenue Share (30% protocol fees)
3. Governance (DAO votes)
4. Collateral (future: borrow USDC)
5. LP Rewards (extra APR for MBX/USDC)

FEE DISTRIBUTION:
├─ USDm LPs: 60%
├─ MBX Stakers: 30%
└─ Buyback & Burn + Treasury: 10%
   ├─ Burn: 7%
   └─ Buffer: 3%

STAKING TIERS:
├─ No lock: 8-15% APY, 0% discount
├─ 3 months: 15-25% APY, 5% discount
├─ 6 months: 25-40% APY, 10% discount
└─ 12 months: 40-70% APY, 15% discount

VESTING:
- Stake MBX → Earn esMBX (non-transferable)
- esMBX vests → MBX (365 days linear)
- Multiplier points (longer stake = higher boost)

EMISSIONS:
- Month 1-3: 100% rate (40-70% APY)
- Month 4-6: 50% rate (25-40% APY)
- Month 7-12: 20% rate (15-25% APY)
- Month 13+: 0% rate (fees only)

BUYBACK & BURN:
- Weekly execution
- TWAP buys (reduce impact)
- Target: 1-7% annual burn @ scale

GOVERNANCE:
- Phase 1 (Year 1): Multisig + community polls
- Phase 2 (Year 2+): Full DAO + veMBX
- Proposals: Fee splits, new markets, treasury
- Voting: 1 MBX = 1 vote (or veMBX boost)

LAUNCH PLAN:
├─ T-4w: Audit & deploy contracts
├─ T-2w: Add liquidity ($500K)
├─ T-1w: Marketing blitz
├─ Day 0: TGE (15M circulating)
├─ Month 1: Bootstrap (40-70% APY)
├─ Month 3: Taper (20-30% APY)
└─ Month 12: Maturity (fees > emissions)

SUCCESS METRICS:
├─ Month 1: $10M vol, $2M TVL, 500 stakers
├─ Month 3: $50M vol, $10M TVL, 2K stakers
├─ Month 6: $150M vol, $50M TVL, 10K stakers
└─ Month 12: $500M vol, $200M TVL, 50K stakers

═══════════════════════════════════════════════════
```

---

## KẾT LUẬN

### **Checklist Hoàn Chỉnh Trước Khi Launch**

**Tokenomics Design**:
- [ ] Total supply decided (fixed/dynamic/hybrid)
- [ ] Allocation pie chart finalized
- [ ] Utility clearly defined (3+ use cases)
- [ ] Vesting schedules set (team, investors, community)
- [ ] Emissions curve planned (taper strategy)
- [ ] Fee distribution model (LP, stakers, burn)
- [ ] Buyback mechanism designed
- [ ] Governance plan (Phase 1 & 2)

**Smart Contracts**:
- [ ] MBX token deployed
- [ ] esMBX deployed
- [ ] Staking contract with tiers
- [ ] Vesting vault
- [ ] Fee router
- [ ] Buyback & burn contract
- [ ] All contracts audited
- [ ] Contracts verified on explorer

**Liquidity**:
- [ ] Initial liquidity added (DEX)
- [ ] LP tokens locked
- [ ] Staking rewards funded
- [ ] Treasury funded
- [ ] Multisig setup (3/5 or 5/7)

**Marketing & Community**:
- [ ] Tokenomics doc published (GitBook)
- [ ] Whitepaper/litepaper
- [ ] Website live
- [ ] Twitter/Discord active
- [ ] KOL partnerships secured
- [ ] Trading competitions planned
- [ ] Referral program ready

**Monitoring**:
- [ ] Analytics dashboard (Dune/custom)
- [ ] Price feeds integrated
- [ ] Alert system (Telegram/Discord bot)
- [ ] Admin panel for team
- [ ] Backup plans (circuit breakers, pause)

---

### **Resources & Further Reading**

**Frameworks**:
- BlackTokenomics: https://blacktokenomics.com
- ThreeSigma Research: https://threesigma.xyz
- Tokenomics.net: https://tokenomics.net

**Tools**:
- Vesting: Sablier, Hedgey
- Analytics: Dune, The Graph, Nansen
- Governance: Snapshot, Tally
- Audits: CertiK, Trail of Bits, OpenZeppelin

**Case Studies**:
- GMX Docs: https://docs.gmx.io
- Curve Docs: https://resources.curve.fi
- Gains Network: https://gains.trade/docs

**Smart Contract Libraries**:
- OpenZeppelin: https://docs.openzeppelin.com
- Solidity by Example: https://solidity-by-example.org

---

### **Final Words**

Tokenomics là **trái tim** của mọi DeFi protocol. Nó không chỉ là pie chart hay emissions schedule, mà là **alignment of incentives** giữa users, investors, team, và protocol.

**Nguyên tắc vàng**:
1. **Utility > Speculation**: Token phải có real use case.
2. **Real Yield > Ponzi**: Fees từ usage, không phải chỉ mint.
3. **Long-term > Short-term**: Vesting, taper, burn.
4. **Transparent > Opaque**: Publish everything, open-source contracts.
5. **Iterate > Rigid**: Monitor metrics, adjust dynamically.

**MegaBanX có nền tảng tốt**:
- Fixed supply (100M) → Scarcity
- Real yield (30% fees) → Sustainable
- esMBX vesting → Anti-dump
- Buyback & burn (7%) → Deflationary
- Tiered staking → Gamification

**Bước tiếp theo**: Deploy testnet, gather feedback, iterate, audit, launch! 🚀

Good luck với MegaBanX! Nếu cần code cụ thể hoặc consult chi tiết hơn, ping anytime.

---

**END OF GUIDE**  
**Version**: 1.0  
**Last Updated**: 09/02/2026  
**Author**: Perplexity AI + Community Research
