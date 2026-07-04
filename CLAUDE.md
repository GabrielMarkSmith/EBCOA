# EBCOA System Architecture & Implementation Rules (`CLAUDE.md`)

## 1. System Overview & Strategic Objective
**EBCOA** ("Event Based Contracts On Anything") is a Web2.5 Peer-to-Peer event/sports wagering application deployed on an Ethereum Layer 2 (L2) network (Target: `Base Mainnet`).

To bypass multi-million dollar regulatory hurdles (FinCEN MSB registration, 50-state Money Transmitter Licenses), EBCOA adopts a strict **"Shield Architecture"** where the platform company never has custody of digital assets, never holds user private keys, and never touches the traditional fiat flow of funds. The platform company acts strictly as a **Software UI Provider** and a **First-Party Oracle**.

---

## 2. Target Architecture Stacks (The Shield Architecture)

### Tier 1: User Identity & Wallets (The Custody Shield)
* **Stack Requirement:** Coinbase Developer Platform (CDP) Ecosystem / Embedded Wallets (WaaS).
* **User Experience:** Users log in using traditional social accounts (Google, Apple ID, Email). 
* **Backend Mechanics:** The SDK automatically spins up a non-custodial MPC wallet under the hood. No seed phrases are surfaced to the user. User database schemas must link internal profiles to these generated MPC wallet addresses.

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

## 3. Immediate Objective: Backend-First Development Block
The project is executing a strict **Backend-First** workflow. Claude Code must isolate all initial scaffolding, data models, cron schedulers, and API polling mechanics within the `/backend` directory before building or deploying production smart contracts. 

Blockchain interactions should be written using clean interface abstractions or modular stubs (`ethers.js` / `@coinbase/coinbase-sdk`) so they can easily execute transactions and handle user MPC wallet structures seamlessly later.

---

## 4. Technical Blueprint: Custom Backend Oracle Engine
The oracle service runs continuously on a secure cloud instance. It handles two core operational tasks: **Locking Kickoffs** (preventing front-running) and **Executing Settlement** (resolving match outcomes).

### `/backend/src/services/oracle.ts`
```typescript
import { Coinbase } from "@coinbase/coinbase-sdk";
import { ethers } from "ethers";

// Initialize Coinbase Developer Platform Ecosystem
if (process.env.CDP_API_KEY_NAME && process.env.CDP_API_KEY_SECRET) {
    Coinbase.configure({ 
        apiKeyName: process.env.CDP_API_KEY_NAME, 
        privateKey: process.env.CDP_API_KEY_SECRET 
    });
}

const PROVIDER_URL = process.env.L2_PROVIDER_URL || "http://localhost:8545"; 
const CONTRACT_ADDRESS = process.env.EBCOA_MANAGER_ADDRESS || ethers.ZeroAddress;
const ORACLE_PRIVATE_KEY = process.env.ORACLE_PRIVATE_KEY || ethers.Wallet.createRandom().privateKey;
const SPORTS_API_KEY = process.env.SPORTS_API_KEY || "MOCK_KEY";

const provider = new ethers.JsonRpcProvider(PROVIDER_URL);
const wallet = new ethers.Wallet(ORACLE_PRIVATE_KEY, provider);

// Expected contract ABI layout for stubbing interaction layers
const contractAbi = [
    "function settleWager(uint256 _wagerId, address _winner) external",
    "function wagers(uint256 _wagerId) external view returns (address creator, address opponent, address tokenAddress, uint256 stakeAmount, uint256 gameId, uint8 status, address winner)",
    "function registerGameDeadline(uint256 _gameId, uint256 _lockTime) external"
];
const ebcoaContract = new ethers.Contract(CONTRACT_ADDRESS, contractAbi, wallet);

/**
 * Pre-emptively locks a game's state before kickoff to block malicious front-running.
 */
export async function lockGameKickoff(sportsApiGameId: number, kickoffTimestamp: number) {
    try {
        console.log(`[ORACLE] Registering kickoff deadline for Game ${sportsApiGameId} at timestamp: ${kickoffTimestamp}`);
        
        // STUB/EXECUTE: In development, log this action or call the local node
        if (CONTRACT_ADDRESS !== ethers.ZeroAddress) {
            const tx = await ebcoaContract.registerGameDeadline(sportsApiGameId, kickoffTimestamp);
            await tx.wait();
            console.
