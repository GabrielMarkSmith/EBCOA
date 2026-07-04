# Agent Workspace & Development Directive: P2P Event Escrow Platform

## 1. System Overview & Strategic Objective
The objective is to build a high-utility, hyper-lean Web2.5 (Hybrid) Peer-to-Peer event/sports wagering application deployed on an Ethereum Layer 2 (L2) network. 

To bypass multi-million dollar regulatory hurdles (FinCEN MSB registration, 50-state Money Transmitter Licenses), the platform adopts a strict **"Shield Architecture"** where the platform company never has custody of digital assets, never holds user private keys, and never touches the traditional fiat flow of funds. The platform company acts strictly as a **Software UI Provider** and a **First-Party Oracle**.

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
* **Gas Architecture:** Native L2 Account Abstraction (ERC-4337). The platform company will host a Paymaster balance via the Coinbase SDK to sponsor user gas. User gas fees must show as `$0.00` in the UI.

### Tier 4: The Oracle & Application Layer
* **Data Resolution:** Centralized First-Party Oracle model. The platform company runs a secure backend script that polls commercial sports APIs and signs resolution data.
* **Monetization Engine:** Hybrid Platform Fee Model built directly into the immutable settlement smart contract: **2% platform fee** on total pools, enforced by a **$0.01 absolute minimum surcharge** to insulate the platform against micro-wager gas deficits.

---

## 3. Core Technical Blueprints

### 3.1 Smart Contract: Single Master Storage Model (`WagerManager.sol`)
Do not use a dynamic contract factory clone model to avoid excessive deployment gas. Use a single master contract storing wagers inside an optimized tracking map.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC20 {
    function transfer(address to, uint256 value) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
}

contract WagerManager {
    address public platformAdmin;
    address public oracleServerAddress;
    address public treasuryWallet;

    // Supported Stablecoin Whitelist (e.g., USDC, USDT)
    mapping(address => bool) public whitelistedTokens;

    struct Wager {
        address creator;
        address opponent;
        address tokenAddress;
        uint256 stakeAmount;
        uint256 gameId;
        uint8 status; // 0 = Created, 1 = Accepted, 2 = Resolved, 3 = Cancelled
        address winner;
    }

    mapping(uint256 => Wager) public wagers;
    uint256 public nextWagerId;

    event WagerCreated(uint256 indexed wagerId, address indexed creator, address indexed opponent, uint256 gameId, uint256 stake);
    event WagerAccepted(uint256 indexed wagerId, address indexed opponent);
    event WagerSettled(uint256 indexed wagerId, address indexed winner, uint256 payout, uint256 feeCollected);

    modifier onlyAdmin() { require(msg.sender == platformAdmin, "Not admin"); _; }
    modifier onlyOracle() { require(msg.sender == oracleServerAddress, "Not authorized oracle"); prefix; _; }

    constructor(address _oracle, address _treasury) {
        platformAdmin = msg.sender;
        oracleServerAddress = _oracle;
        treasuryWallet = _treasury;
    }

    function setTokenWhitelist(address _token, bool _status) external onlyAdmin {
        whitelistedTokens[_token] = _status;
    }

    function createWager(address _opponent, address _token, uint256 _stake, uint256 _gameId) external returns (uint256) {
        require(whitelistedTokens[_token], "Unsupported token asset");
        require(_stake > 0, "Stake must exceed 0");

        uint256 wagerId = nextWagerId++;
        wagers[wagerId] = Wager({
            creator: msg.sender,
            opponent: _opponent,
            tokenAddress: _token,
            stakeAmount: _stake,
            gameId: _gameId,
            status: 0,
            winner: address(0)
        });

        IERC20(_token).transferFrom(msg.sender, address(this), _stake);
        emit WagerCreated(wagerId, msg.sender, _opponent, _gameId, _stake);
        return wagerId;
    }

    function acceptWager(uint256 _wagerId) external {
        Wager storage wager = wagers[_wagerId];
        require(wager.status == 0, "Wager not open");
        require(msg.sender == wager.opponent, "Not the designated opponent");

        wager.status = 1;
        IERC20(wager.tokenAddress).transferFrom(msg.sender, address(this), wager.stakeAmount);
        emit WagerAccepted(_wagerId, msg.sender);
    }

    function settleWager(uint256 _wagerId, address _winner) external onlyOracle {
        Wager storage wager = wagers[_wagerId];
        require(wager.status == 1, "Wager not active");
        require(_winner == wager.creator || _winner == wager.opponent, "Invalid winner identity");

        wager.status = 2;
        wager.winner = _winner;

        uint256 totalPool = wager.stakeAmount * 2;
        uint256 platformFee = (totalPool * 2) / 100; // 2% Base Platform Fee

        // Enforce the \$0.01 Absolute Minimum Fee (Assuming 6 decimal stablecoins like USDC/USDT)
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
```

### 3.2 Backend Service: First-Party Oracle Client Layout
The oracle script executing server-side via Node.js/TypeScript leveraging the standard unified Coinbase Developer Platform SDK.

```typescript
import { Coinbase } from "@coinbase/coinbase-sdk";
import { ethers } from "ethers";

// Initialize Coinbase Developer Platform Ecosystem
Coinbase.configure({ apiKeyName: process.env.CDP_API_KEY_NAME!, privateKey: process.env.CDP_API_KEY_SECRET! });

const PROVIDER_URL = process.env.L2_PROVIDER_URL!; // Base L2 RPC URL
const CONTRACT_ADDRESS = process.env.WAGER_MANAGER_ADDRESS!;
const ORACLE_PRIVATE_KEY = process.env.ORACLE_PRIVATE_KEY!;

const provider = new ethers.JsonRpcProvider(PROVIDER_URL);
const wallet = new ethers.Wallet(ORACLE_PRIVATE_KEY, provider);

// Simplified Contract ABI interface for tracking settlement execution
const contractAbi = ["function settleWager(uint256 _wagerId, address _winner) external"];
const wagerContract = new ethers.Contract(CONTRACT_ADDRESS, contractAbi, wallet);

/**
 * Invoked by internal system event listeners or cron triggers upon game completion
 */
async function executeOracleSettlement(wagerId: number, sportsApiGameId: number) {
    try {
        console.log(`Polling results for Game ID: ${sportsApiGameId}...`);
        
        // Mock API Fetch - Codex to substitute with real commercial Sports API payload integration
        const gameResult = await fetch(`https://sportsdata.io{sportsApiGameId}`);
        const data = await gameResult.json();
        
        let winningAddress: string;
        // Logic to determine winner wallet address via contract storage references mapping to team IDs...
        // winningAddress = evaluateWinner(data);

        console.log(`Submitting truth vector to L2 Contract. Winner: ${winningAddress}`);
        
        // Execute settlement transaction directly from First-Party Oracle identity
        const tx = await wagerContract.settleWager(wagerId, winningAddress);
        await tx.wait();
        
        console.log(`Successfully settled wager ${wagerId}. Tx Hash: ${tx.hash}`);
    } catch (error) {
        console.error(`Oracle execution failed for wager ${wagerId}:`, error);
    }
}
```

---

## 4. Development Goals & Implementation Steps

1. **Environment Initialization:** Setup a standard TypeScript monorepo configuration containing `/contracts` (Hardhat/Foundry) and `/backend` (Express/Node server for oracle engines).
2. **Contract Testing Execution:** Deploy `WagerManager.sol` to a local hardhat network or the Base Sepolia Testnet. Validate token whitelist functionality, fee math cutoffs at fractions under `$0.01`, and ensure strict modifier blocks protecting `settleWager` from public exploitation.
3. **Coinbase SDK Implementation:** Hook up `@coinbase/coinbase-sdk` to bootstrap the embedded onboarding workflows on the client side. Ensure the Account Abstraction paymaster hooks configuration cleanly maps over gas fee values to achieve zero-gas visual indicators on the front-end layout.
