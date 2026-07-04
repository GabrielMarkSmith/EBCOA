# EBCOA Development Directive & System Architecture (`CLAUDE.md`)

## 1. System Overview & Strategic Objective
The objective is to build **EBCOA** ("Event Based Contracts On Anything"), a high-utility, hyper-lean Web2.5 (Hybrid) Peer-to-Peer event and sports wagering application deployed on an Ethereum Layer 2 (L2) network. 

To bypass regulatory hurdles (FinCEN MSB registration, 50-state Money Transmitter Licenses), EBCOA adopts a strict **"Shield Architecture"** where the platform company never has custody of digital assets, never holds user private keys, and never touches traditional fiat flow of funds. The platform company acts strictly as a **Software UI Provider** and a **First-Party Oracle**.

---

## 2. Target Architecture Stacks

### Tier 1: User Identity & Wallets (The Custody Shield)
* **Stack Requirement:** Coinbase Developer Platform (CDP) Ecosystem / Embedded Wallets (WaaS).
* **User Experience:** Users log in using traditional social accounts (Google, Apple ID, Email). 
* **Backend Mechanics:** The SDK automatically spins up a non-custodial MPC wallet under the hood. No seed phrases are surfaced to the user.

### Tier 2: Banking, Fiat On/Off-Ramps (The Cash Shield)
* **Stack Requirement:** Coinbase On-Ramp API / CDP Integration.
* **Mechanics:** All fiat-to-stablecoin exchanges are routed through Coinbase's compliant banking rails via an embedded iframe/SDK widget. 

### Tier 3: Core Blockchain Layer (The Infrastructure Shield)
* **Network Target:** Ethereum Layer 2 (Primary Target: `Base Mainnet`).
* **Supported Assets:** Stablecoins only. Rigidly limited to Native L2 **USDC** and **USDT**. 
* **Gas Architecture:** Native L2 Account Abstraction (ERC-4337). EBCOA will host a Paymaster balance via the Coinbase SDK to sponsor user gas. User gas fees must show as **$0.00** in the UI.

### Tier 4: The Oracle & Application Layer
* **Data Resolution:** Centralized First-Party Oracle model. The EBCOA backend runs a secure, stateful service that polls commercial sports/event APIs, validates data integrity, protects against stale data manipulation, and signs resolution vectors.
* **Monetization Engine:** Hybrid Platform Fee Model built directly into the immutable settlement smart contract: **2% platform fee** on total pools, enforced by a **$0.01 absolute minimum surcharge** to insulate the platform against micro-wager gas deficits.

---

## 3. Core Technical Blueprints

### 3.1 Smart Contract: Single Master Storage Model (`EbcoaManager.sol`)
Do not use a dynamic contract factory clone model to avoid excessive deployment gas. Use a single master contract storing wagers inside an optimized tracking map.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC20 {
    function transfer(address to, uint256 value) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
}

contract EbcoaManager {
    address public platformAdmin;
    address public oracleServerAddress;
    address public treasuryWallet;

    // Supported Stablecoin Whitelist (e.g., USDC, USDT)
    mapping(address => bool) public whitelistedTokens;

    enum WagerStatus { Created, Accepted, Resolved, Cancelled }

    struct Wager {
        address creator;
        address opponent;
        address tokenAddress;
        uint256 stakeAmount;
        uint256 gameId;
        WagerStatus status;
        address winner;
    }

    mapping(uint256 => Wager) public wagers;
    uint256 public nextWagerId;

    // Oracle Verification Mapping: tracking external APIs safely
    // gameId => gameEndTimeStamp (0 means unmapped/not verified yet)
    mapping(uint256 => uint256) public verifiedGameDeadlines;

    event WagerCreated(uint256 indexed wagerId, address indexed creator, address indexed opponent, uint256 gameId, uint256 stake);
    event WagerAccepted(uint256 indexed wagerId, address indexed opponent);
    event WagerSettled(uint256 indexed wagerId, address indexed winner, uint256 payout, uint256 feeCollected);
    event GameDeadlineSet(uint256 indexed gameId, uint256 lockTime);

    modifier onlyAdmin() { 
        require(msg.sender == platformAdmin, "Not admin"); 
        _; 
    }
    
    modifier onlyOracle() { 
        require(msg.sender == oracleServerAddress, "Not authorized oracle"); 
        _; 
    }

    constructor(address _oracle, address _treasury) {
        platformAdmin = msg.sender;
        oracleServerAddress = _oracle;
        treasuryWallet = _treasury;
    }

    function setTokenWhitelist(address _token, bool _status) external onlyAdmin {
        whitelistedTokens[_token] = _status;
    }

    /**
     * @dev Allows the Oracle engine to register structural lock timestamps for game IDs.
     * Prevents front-running bets on games that have already started or concluded.
     */
    function registerGameDeadline(uint256 _gameId, uint256 _lockTime) external onlyOracle {
        require(verifiedGameDeadlines[_gameId] == 0, "Game deadline already registered");
        verifiedGameDeadlines[_gameId] = _lockTime;
        emit GameDeadlineSet(_gameId, _lockTime);
    }

    function createWager(address _opponent, address _token, uint256 _stake, uint256 _gameId) external returns (uint256) {
        require(whitelistedTokens[_token], "Unsupported token asset");
        require(_stake > 0, "Stake must exceed 0");
        
        // Anti-Frontrunning Guard
        uint256 deadline = verifiedGameDeadlines[_gameId];
        require(deadline == 0 || block.timestamp < deadline, "Target game has already commenced");

        uint256 wagerId = nextWagerId++;
        wagers[wagerId] = Wager({
            creator: msg.sender,
            opponent: _opponent,
            tokenAddress: _token,
            stakeAmount: _stake,
            gameId: _gameId,
            status: WagerStatus.Created,
            winner: address(0)
        });

        IERC20(_token).transferFrom(msg.sender, address(this), _stake);
        emit WagerCreated(wagerId, msg.sender, _opponent, _gameId, _stake);
        return wagerId;
    }

    function acceptWager(uint256 _wagerId) external {
        Wager storage wager = wagers[_wagerId];
        require(wager.status == WagerStatus.Created, "Wager not open");
        require(msg.sender == wager.opponent, "Not the designated opponent");
        
        // Anti-Frontrunning Guard
        uint256 deadline = verifiedGameDeadlines[wager.gameId];
        require(deadline == 0 || block.timestamp < deadline, "Target game has already commenced");

        wager.status = WagerStatus.Accepted;
        IERC20(wager.tokenAddress).transferFrom(msg.sender, address(this), wager.stakeAmount);
        emit WagerAccepted(_wagerId, msg.sender);
    }

    function settleWager(uint256 _wagerId, address _winner) external onlyOracle {
        Wager storage wager = wagers[_wagerId];
        require(wager.status == WagerStatus.Accepted, "Wager not active");
        require(_winner == wager.creator || _winner == wager.opponent, "Invalid winner identity");

        wager.status = WagerStatus.Resolved;
        wager.winner = _winner;

        uint256 totalPool = wager.stakeAmount * 2;
        uint256 platformFee = (totalPool * 2) / 100; // 2% Base Platform Fee

        // Enforce the $0.01 Absolute Minimum Fee (Assuming 6 decimal stablecoins like USDC/USDT)
        // 0.01 Token = 10,000 minimal units
        if (platformFee < 10000) {
            platformFee = 10000;
        }

        uint256 payoutAmount = totalPool - platformFee;

        IERC20(wager.tokenAddress).transfer(treasuryWallet, platformFee);
        IERC20(wager.tokenAddress).transfer(_winner, payoutAmount);

        emit WagerSettled(_wagerId, _winner, payoutAmount, platformFee);
    }
}
