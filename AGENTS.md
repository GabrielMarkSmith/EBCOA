# EBCOA (Event Based Contracts on Anything)

## 1. System Overview & Strategic Objective

The objective is to build a peer-to-peer wagering and social platform on an Ethereum Layer 2 (L2) network. 

### Wagering 

### Social Platform 

 The platform should allow users to create a custom social profile. Users should also be allowed to create a custom bio and also add their favorite teams to be displayed on their profile page. 

 The vision for the social aspect of the platform is to allow people to add "buddies", which effectively acts as a "friends" list. Buddies will be able to directly message each other in a direct messaging platform
 
 to connect with friends and meet new peopl create event based contracts on anything they want. /sports wagering application deployed on an Ethereum Layer 2 (L2) network. 

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
