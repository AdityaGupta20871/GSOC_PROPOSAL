# Google Summer of Code 2025

## Project Proposal - BenefactionPlatform-Ergo: Expanding Fundraising Capabilities on Ergo

### Contact Information
Name: [Aditya Gupta]

Email: [adityagupta20871@gmail.com]

GitHub: [https://github.com/AdityaGupta20871]

LinkedIn: [https://www.linkedin.com/in/aditya-gupta-732857233/]

Discord: [Aditya_Gupta]

### Education
College Name: [Maharaja Agrasen Institute of Technology]

Degree: [B.Tech in Computer Science and Engineering]

Expected Graduation Year: [2025]

CGPA/GPA: [8.656]

### Commitment
**How many hours will you work per week on your GSoC project?**
I anticipate being able to dedicate approximately 35-40 hours per week to this project during the GSoC period.

**Do you have any commitments during the GSoC Period?**
[I currently have no major commitments that would interfere with my GSoC participation. While I am preparing for the GRE to pursue graduate studies in research abroad, I can fully commit to the GSoC schedule. If selected, I would prioritize this opportunity as it aligns perfectly with my research aspirations and would be an invaluable addition to my academic profile.]

## Chosen Project: BenefactionPlatform-Ergo - Multi-Token Support and EVM Feature Parity

### 1. Introduction: Enhancing Ergo's Fundraising Ecosystem

The Bene Fundraising Platform on Ergo blockchain provides a unique proof-of-funding mechanism through the APT/PFT token system, enabling transparent project funding. While the EVM version of Bene supports multiple wallets, networks, and token types, the Ergo implementation currently has limitations. This proposal aims to enhance the Ergo version of Bene by implementing multi-token support (especially stablecoins) and achieving greater feature parity with the EVM version while considering specific Ergo-specific needs and capabilities.

### 2. Technical Gap Analysis

Based on an analysis of the codebase and Bruno's vision, I've identified the following key areas for improvement in order of priority:

1. **Limited Token Support**: Currently, the Ergo implementation only supports ERG for contributions, while the EVM version supports multiple tokens including stablecoins.
2. **Wallet Integration Framework**: Need for a comprehensive multi-wallet integration system similar to the EVM version.
3. **Referral System**: Implementation of a referral mechanism to incentivize project marketing while preventing exploitation.
4. **Auxiliary Exchange Functionality**: Enhanced token exchange capabilities between APTs and PFTs.
5. **Analytics Dashboard**: Development of project performance tracking and visualization tools.
6. **Platform Unification**: Reducing fragmentation between EVM and Ergo implementations to minimize duplicate development efforts.
7. **Bounty Platform Integration**: Potential to incorporate elements from the Bountiful proposal for decentralized bounty management.


### 2.1. Current State Analysis

#### Technology Stack

#### Frontend
- **SvelteKit**: Modern web framework for building Svelte applications
- **TypeScript**: Typed superset of JavaScript for improved developer experience
- **Tailwind CSS**: Utility-first CSS framework for rapid UI development

#### Blockchain Interaction
- **Fleet SDK**:
  - `@fleet-sdk/compiler`: For compiling ErgoScript contracts
  - `@fleet-sdk/core`: Core libraries for Ergo blockchain interaction

#### Wallet Integration
- **Nautilus Wallet**: Via ergoConnector for user authentication and transaction signing

#### Smart Contracts
- **ErgoScript**: Versions 1.0 and 1.1
  - UTXO-based smart contract language specific to the Ergo blockchain


## 3. Proposed Technical Implementations

To address these gaps and align with Bruno's vision, I propose implementing the following features in priority order:

## 3.1 Multi-Token Support With Emphasis on Stablecoins

**Technical Challenge:** The current contract design primarily handles ERG as the contribution currency.

**Solution:** Implement a simplified approach that allows the project creator to select a single base token (which can be ERG or any other token, including stablecoins) per project while avoiding unnecessary complexity.

**Theoretical Framework:**

1. **Single Base Token Model**
   - Instead of implementing support for multiple base tokens simultaneously, we adopt a simpler "single base token per project" approach
   - Each project defines one base token (BT), which can be ERG or any Ergo token including stablecoins
   - The exchange rate is defined as BT/PFT (Base Token to Proof of Funding Token)
   - Users contribute using only this designated base token

2. **Register Optimization**
   - Extend the existing R7 register to store both:
     - The base exchange rate (as it currently does)
     - The base token ID (empty string represents ERG)
   - This approach avoids adding additional registers while maintaining backward compatibility

3. **Transaction Flow Enhancement**
   - For ERG projects: Transaction flow remains identical to current implementation
   - For token projects: The contract validates that input tokens match the specified base token ID
   - Contribution amount validation is based on token quantity rather than ERG value

4. **User Experience Considerations**
   - Project creators can select their preferred base token during project creation
   - Contributors can acquire the project's base token from any DEX if they don't already hold it
   - The UI clearly communicates which token is required for contribution

**Pseudocode Implementation:**

1. **Contract Modification (contract_v1_1.es):**
   ```ergoscript
   // Extract base token ID and exchange rate from R7
   val baseTokenId = SELF.R7[Coll[Byte]].get.slice(8, SELF.R7[Coll[Byte]].get.size)
   val baseExchangeRate = SELF.R7[Long].get
   
   // Check if base token is ERG
   val isErgBaseToken = baseTokenId.size == 0
   
   // Validate correct exchange based on token type
   val correctExchange = if (isErgBaseToken) {
     // Original ERG logic
     val deltaValueAdded = OUTPUTS(0).value - selfValue
     deltaValueAdded == deltaTokenRemoved * baseExchangeRate
   } else {
     // Token contribution logic
     val contributionAmount = INPUTS(1).tokens.filter(_._1 == baseTokenId).fold(0L, (acc, t) => acc + t._2)
     contributionAmount == deltaTokenRemoved * baseExchangeRate
   }
   ```

2. **Contract Parameters Creation (contract.ts):**
   ```typescript
   function createContractParameters(config) {
     // Combine exchange rate with base token ID for R7
     const r7Value = config.baseTokenId 
       ? bytesToHex(longToByteArray(config.baseExchangeRate)) + config.baseTokenId
       : config.baseExchangeRate;
     
     return {
       // Other registers remain the same
       R7: r7Value
     };
   }
   ```

3. **Transaction Building (buy_refund.ts):**
   ```typescript
   function buildBuyTransaction(contractBox, userAddress, amount) {
     // Extract base token info from R7
     const baseExchangeRate = BigInt(contractBox.additionalRegisters.R7.value);
     const baseTokenId = contractBox.additionalRegisters.R7.serializedValue.slice(16); // Skip the 8-byte rate
     
     // Calculate token value
     const tokenValue = amount * baseExchangeRate;
     
     // Build different transaction based on token type
     if (baseTokenId === "") {
       // Build ERG transaction (existing logic)
     } else {
       // Build token transaction
       return new TransactionBuilder(height)
         .from(contractBox)
         .from(userBox) // Contains the tokens
         .to(new OutputBuilder(contractBox.value + tokenRequirement, contractBox.address)
             .addTokens({ tokenId: contractBox.tokens[0].tokenId, amount: -amount }))
         .to(new OutputBuilder(minBoxValue, userAddress)
             .addTokens({ tokenId: contractBox.tokens[0].tokenId, amount }))
         .sendChangeTo(userAddress);
     }
   }
   ```

4. **Project Creation (submit.ts):**
   ```typescript
   function createProjectWithBaseToken(projectConfig) {
     // Add base token selection to project creation
     const baseTokenId = projectConfig.baseTokenId || ""; // Empty for ERG
     const baseExchangeRate = projectConfig.baseExchangeRate;
     
     // Create contract with modified R7
     const contract = compileContract({
       // Other parameters
       baseTokenId,
       baseExchangeRate
     });
     
     // Create and submit transaction
   }
   ```

5. **Token Selection UI:**
   ```typescript
   // Simple token selector component
   function TokenSelector({ onSelect }) {
     const [tokens, setTokens] = useState([]);
     
     useEffect(() => {
       // Fetch user's tokens from wallet
       async function loadTokens() {
         const address = await ergoConnector.nautilus.getChangeAddress();
         const balance = await ergo.get_balance(address);
         setTokens(balance.tokens);
       }
       
       loadTokens();
     }, []);
     
     return (
       <div>
         <select onChange={(e) => onSelect(e.target.value)}>
           <option value="">ERG (Native)</option>
           {tokens.map(token => (
             <option key={token.tokenId} value={token.tokenId}>
               {token.name || token.tokenId.substring(0, 8)}
             </option>
           ))}
         </select>
       </div>
     );
   }
   ```

**Key Advantages:**
1. Significantly reduces contract complexity
2. Eliminates the need for data inputs to determine token exchange rates
3. No separate token registry contract required
4. Provides flexibility for project creators to use stablecoins or other tokens
5. Reduces on-chain transaction complexity and costs
6. Maintains a consistent user experience regardless of base token

**Implementation References:**
1. For token handling: [Fleet SDK Transaction Building](https://fleet-sdk.github.io/docs/transaction-building)
2. For token selection UI: [Nautilus Wallet GitHub](https://github.com/nautls/nautilus-wallet)
3. For DEX integration: [ErgoDEX SDK](https://github.com/ergolabs/ergo-dex-sdk-js)

**Files that Need Modification:**
1. `contracts/bene_contract/contract_v1_1.es` - Update contract to handle configurable base token
2. `src/lib/ergo/contract.ts` - Update contract compilation and parameter handling
3. `src/lib/ergo/actions/buy_refund.ts` - Modify transaction building to handle token inputs
4. `src/lib/ergo/actions/submit.ts` - Update project creation flow
5. UI components for token selection


## 4. Multi-Wallet Integration and Network Support

**Technical Challenge:** The EVM version supports multiple wallets and networks, while the Ergo version has limited wallet support.

**Solution:** Create a modular wallet integration framework for Ergo that supports multiple wallet providers.

**Technical Implementation:**
1. Develop a wallet adapter interface:
   ```typescript
   // Wallet adapter interface
   interface ErgoWalletAdapter {
     // Get wallet addresses
     getAddresses(): Promise<string[]>;
     
     // Get wallet balance
     getBalance(address: string): Promise<{
       nanoErgs: BigNumber;
       tokens: Array<{ tokenId: string, amount: BigNumber }>;
     }>;
     
     // Sign transaction
     signTransaction(unsignedTx: UnsignedTransaction): Promise<SignedTransaction>;
     
     // Submit transaction
     submitTransaction(signedTx: SignedTransaction): Promise<string>;
     
     // Events
     on(event: 'addressChanged' | 'networkChanged', callback: Function): void;
   }
   ```

2. Implement wallet-specific adapters:
   ```typescript
   // Nautilus Wallet Adapter
   class NautilusWalletAdapter implements ErgoWalletAdapter {
     private readonly _ergoConnector = window.ergoConnector?.nautilus;
     
     async connect(): Promise<boolean> {
       if (!this._ergoConnector) {
         throw new Error('Nautilus wallet not available');
       }
       return await this._ergoConnector.connect();
     }
     
     async getAddresses(): Promise<string[]> {
       if (!(await this.isConnected())) {
         throw new Error('Wallet not connected');
       }
       return await window.ergo.get_used_addresses();
     }
     
     async signTransaction(unsignedTx: UnsignedTransaction): Promise<SignedTransaction> {
       return await window.ergo.sign_tx(unsignedTx);
     }
     
     // Full implementation with EIP-12 protocol support
   }
   
   // SAFEW Wallet Adapter
   class SafewWalletAdapter implements ErgoWalletAdapter {
     private readonly _ergoConnector = window.ergoConnector?.safew;
     
     async connect(): Promise<boolean> {
       if (!this._ergoConnector) {
         throw new Error('SAFEW wallet not available');
       }
       // SAFEW implements the EIP-12 protocol like Nautilus
       return await this._ergoConnector.connect();
     }
     
     // Similar implementation to NautilusWalletAdapter with SAFEW-specific handling
     // SAFEW fully implements the EIP-12 specification in accordance with:
     // https://github.com/Emurgo/Emurgo-Research/blob/master/ergo/EIP-0012.md
   }
   
   // Mobile Wallet Adapter (ErgoPay)
   class MobileWalletAdapter implements ErgoWalletAdapter {
     private _connectionUrl: string;
     private _transactionId?: string;
     
     constructor(connectionUrl: string) {
       this._connectionUrl = connectionUrl;
     }
     
     async getAddresses(): Promise<string[]> {
       // Use ErgoPay protocol to get wallet addresses
       const response = await fetch(`${this._connectionUrl}/addresses`);
       const data = await response.json();
       return data.addresses;
     }
     
     async signTransaction(unsignedTx: UnsignedTransaction): Promise<SignedTransaction> {
       // Create transaction request for ErgoPay
       const txRequest = {
         reducedTx: unsignedTx,
         sender: this._selectedAddress,
         callback: window.location.origin + '/wallet/callback'
       };
       
       // Generate ErgoPay URL
       const encodedTx = encodeURIComponent(JSON.stringify(txRequest));
       const ergoPayUrl = `ergopay://${this._connectionUrl}/transaction/sign?tx=${encodedTx}`;
       
       // For mobile browser integration
       if (this._isMobileDevice()) {
         window.location.href = ergoPayUrl;
       }
       
       // For desktop use, generate QR code with ergoPayUrl
       return this._waitForTransaction();
     }
     
     // Implementation for QR code and deep linking support
   }
   ```

3. Create a unified connection management system:
   ```typescript
   enum WalletType {
     Nautilus = 'nautilus',
     Safew = 'safew',
     Mobile = 'mobile',
     // Future wallet types can be added here
   }
   
   interface WalletInfo {
     type: WalletType;
     addresses: string[];
     network: Network;
     connected: boolean;
   }
   
   class ErgoConnectionManager {
     private activeWallet: ErgoWalletAdapter | null = null;
     private networkManager = new ErgoNetworkManager();
     private walletInfo: WalletInfo | null = null;
     
     // Connect to wallet
     async connect(walletType: WalletType): Promise<boolean> {
       try {
         // Create appropriate wallet adapter
         switch (walletType) {
           case WalletType.Nautilus:
             this.activeWallet = new NautilusWalletAdapter();
             break;
           case WalletType.Safew:
             this.activeWallet = new SafewWalletAdapter();
             break;
           case WalletType.Mobile:
             // For mobile wallet, we would need a connection URL
             // This could be provided separately or determined dynamically
             const connectionUrl = 'https://api.benefaction.ergo/ergopay';
             this.activeWallet = new MobileWalletAdapter(connectionUrl);
             break;
           default:
             throw new Error(`Unsupported wallet type: ${walletType}`);
         }
         
         // Connect to wallet
         const connected = await this.activeWallet.connect();
         
         if (connected) {
           // Get addresses
           const addresses = await this.activeWallet.getAddresses();
           
           // Detect network
           const network = await this.networkManager.detectNetwork(this.activeWallet);
           
           // Update wallet info
           this.walletInfo = {
             type: walletType,
             addresses,
             network,
             connected
           };
           
           // Subscribe to address changes
           this.activeWallet.on('addressChanged', async () => {
             if (this.walletInfo) {
               this.walletInfo.addresses = await this.activeWallet!.getAddresses();
             }
           });
         }
         
         return connected;
       } catch (error) {
         console.error('Failed to connect wallet:', error);
         this.disconnect();
         return false;
       }
     }
     
     // Disconnect from wallet
     disconnect(): void {
       this.activeWallet = null;
       this.walletInfo = null;
     }
     
     // Get connected wallet info
     getWalletInfo(): WalletInfo | null {
       return this.walletInfo;
     }
     
     // Get active wallet adapter
     getWallet(): ErgoWalletAdapter | null {
       return this.activeWallet;
     }
   }
   ```

4. Build a modular UI component system for wallet connection:
   ```svelte
   <!-- WalletConnector.svelte -->
   <script lang="ts">
     import { onMount } from 'svelte';
     import { ErgoConnectionManager, WalletType } from '../lib/wallet';
     
     const connectionManager = new ErgoConnectionManager();
     let walletInfo = null;
     let availableWallets = [];
     let connecting = false;
     let error = '';
     
     onMount(async () => {
       // Check which wallets are available by detecting the ergoConnector
       if (window.ergoConnector) {
         if (window.ergoConnector.nautilus) {
           availableWallets.push({
             type: WalletType.Nautilus,
             name: 'Nautilus',
             logo: '/assets/nautilus-logo.svg'
           });
         }
         
         if (window.ergoConnector.safew) {
           availableWallets.push({
             type: WalletType.Safew,
             name: 'SAFEW',
             logo: '/assets/safew-logo.svg'
           });
         }
       }
       
       // Mobile wallet is always available via ErgoPay
       availableWallets.push({
         type: WalletType.Mobile,
         name: 'Mobile Wallet (ErgoPay)',
         logo: '/assets/ergopay-logo.svg'
       });
     });
     
     async function connectWallet(walletType) {
       error = '';
       connecting = true;
       
       try {
         const connected = await connectionManager.connect(walletType);
         if (connected) {
           walletInfo = connectionManager.getWalletInfo();
         } else {
           error = 'Failed to connect to wallet';
         }
       } catch (e) {
         error = e.message || 'Failed to connect to wallet';
       } finally {
         connecting = false;
       }
     }
     
     function disconnectWallet() {
       connectionManager.disconnect();
       walletInfo = null;
     }
   </script>

   <div class="wallet-connector">
     {#if !walletInfo}
       <h3>Connect Wallet</h3>
       {#if error}
         <div class="error">{error}</div>
       {/if}
       
       <div class="wallet-list">
         {#each availableWallets as wallet}
           <button 
             class="wallet-button" 
             on:click={() => connectWallet(wallet.type)}
             disabled={connecting}
           >
             <img src={wallet.logo} alt={wallet.name} />
             <span>{wallet.name}</span>
           </button>
         {/each}
       </div>
     {:else}
       <div class="wallet-info">
         <h3>Connected Wallet</h3>
         <div>
           <strong>Type:</strong> {walletInfo.type}
         </div>
         <div>
           <strong>Network:</strong> {walletInfo.network.networkType}
         </div>
         <div>
           <strong>Addresses:</strong>
           <select>
             {#each walletInfo.addresses as address}
               <option>{address}</option>
             {/each}
           </select>
         </div>
         <button on:click={disconnectWallet}>Disconnect</button>
       </div>
     {/if}
   </div>
   ```

**Wallet Integration Flow:**
```
[Wallet Selection UI] → [Connect to Selected Wallet] → [Network Detection]
        ↓                           ↓                          ↓
[Permission Request] → [Address Retrieval] → [Connection Management]
```

**Supported Wallet Documentation and Integration:**

1. **Nautilus Wallet**
   - [Official Documentation](https://docs.nautiluswallet.com/dapp-connector/api-overview)
   - [GitHub Repository](https://github.com/nautls/nautilus-wallet)
   - Integration Method: Implements EIP-12 protocol through the `ergoConnector.nautilus` interface
   - Browser Support: Chrome, Firefox, Edge, Brave

2. **SAFEW (Simple And Fast Ergo Wallet)**
   - [GitHub Documentation](https://github.com/ThierryM1212/SAFEW)
   - Integration Method: Implements EIP-12 protocol similarly to Nautilus 
   - Browser Support: Chrome, Firefox, Edge, Brave

3. **Mobile Wallet (via ErgoPay Protocol)**
   - [Official ErgoPay Documentation](https://docs.ergoplatform.com/dev/wallet/payments/ergopay/ergo-pay/)
   - [ErgoPay GitHub Sample Implementation](https://github.com/MrStahlfelge/ergopay-payment-portal)
   - Integration Method: QR code/deep linking with transaction signing through mobile wallet app
   - Mobile Support: Android and iOS Ergo Wallet Apps (v1.6+)

This comprehensive multi-wallet integration framework provides flexibility and choice for users while maintaining a consistent interface for the application code, regardless of which wallet provider is being used. The abstraction layer handles the specifics of each wallet's implementation, making it possible to add support for new wallets in the future without major code changes.

## 5. Referral System for Project Marketing

To boost project marketing, I propose implementing a referral system that incentivizes content creators and advertising agencies to promote projects while addressing potential exploitation.

**Core Mechanism:**

1. **Referral Allocation Formula:**
   ```
   referralReward = contributionAmount * referralPercentage
   
   Where:
   - referralPercentage is typically 5% of the project's PFT allocation
   - If no referrer is specified, these tokens remain locked until project completion
   ```

2. **Anti-Exploitation Safeguard:**
   To prevent contributors from naming themselves as referrers, a staking mechanism will be implemented:
   ```
   minStakeRequired = targetMedianContribution * stakeFactor
   
   Where:
   - targetMedianContribution is the expected median contribution
   - stakeFactor is a multiplier (typically 1.5-2x) that makes self-referral economically unattractive
   ```

3. **Referrer Qualification Logic:**
   ```
   isValidReferrer(address) = hasValidStake(address) && !isSelfReferral(address, contributor)
   
   Where:
   - hasValidStake checks if the referrer has staked the minimum required amount
   - isSelfReferral prevents a contributor from using their own address
   ```

4. **Dynamic Referral Rate:**
   To encourage higher stakes and better marketing efforts:
   ```
   effectiveRate = baseRate + min(bonusRate * (stake / baseStake - 1), maxBonus)
   
   Where:
   - baseRate is the standard referral percentage (5%)
   - bonusRate adds incremental percentage based on stake size
   - maxBonus caps the maximum referral percentage (e.g., 10%)
   ```

5. **Implementation Approach:**
   - Store referrer information in Ergo box registers
   - Use a URL parameter system for referral tracking
   - Implement time-based expiration for referral codes
   - Create a verification system that validates stakes on-chain

This system creates proper economic incentives for marketing while preventing abuse. The staking requirement means only serious promoters will participate, as the cost of gaming the system exceeds the potential benefit from self-referral.

Using Ergo's native capabilities, I'll implement this with minimal overhead while ensuring security and transparency for all participants.

## 6. Optimized Blockchain Data Fetching Solutions

## Background
The Ergo blockchain API has limitations in filtering and sorting capabilities:
1. Sorting is restricted to default order and its reverse
2. Register-based filtering is complex since registers like R9 contain different types of information
3. The API doesn't support text searches within register content

## Solutions

### 1. Smart Caching Strategy

**Solution:** Implement a TTL-based (Time-To-Live) caching mechanism that stores API responses in memory with an expiration time. This reduces repeated API calls for the same data and implements background refresh to ensure data freshness without impacting user experience.

**Files to modify:**
- **New File: `src/lib/ergo/cache.ts`**
  ```typescript
  // Create a generic cache class with TTL support
  export class BlockchainCache<T> {
    // Store cache items with expiration timestamps
    // Implement get, set, invalidate methods
    // Add background refresh logic for expiring items
  }

  // Export a singleton instance for projects
  export const projectsCache = new BlockchainCache<Map<string, Project>>();
  ```

- **Modify: `src/lib/ergo/fetch.ts`**
  ```typescript
  // Modify fetch_projects to use cache
  export async function fetch_projects(offset: number = 0): Promise<Map<string, Project>> {
    // Use cache.get with a fetch function as fallback
    return projectsCache.get(`projects-${offset}`, 
      () => fetchProjectsFromBlockchain(offset));
  }
  ```

### 2. Client-Side Indexing and Filtering

**Solution:** Create an in-memory index of fetched projects to enable powerful text search, sorting, and filtering operations entirely on the client side. This approach bypasses API limitations by performing post-processing on already retrieved data.

**Files to modify:**
- **New File: `src/lib/ergo/memoryIndex.ts`**
  ```typescript
  // Create an in-memory search index
  export class MemoryIndex {
    // Store tokenized text for fast search
    // Implement methods for search, filter, and sort
    // Optimize for performance with weighted fields
  }
  ```

### 3. Progressive Loading Pattern

**Solution:** Implement an infinite-scroll pattern that loads data in small batches and renders only the visible elements. This reduces memory consumption and improves performance when dealing with large datasets.

**Files to modify:**
- **New/Modify: `src/routes/+page.svelte`** (or your project list view)
  ```typescript
  // Implement virtual scrolling with IntersectionObserver
  // Load projects in batches when user scrolls
  // Apply client-side filtering and sorting
  ```

### 4. Integration Strategy

**Theory:** Combine the above approaches into a unified data access layer that handles caching, filtering, and progressive loading. This creates a seamless experience that feels like a native app despite API limitations.

**Files to modify:**
- **New File: `src/lib/ergo/dataAccess.ts`**
  ```typescript
  // Create a facade for data operations
  export async function getFilteredProjects(options) {
    // Use cache for raw data
    // Apply in-memory filtering
    // Support pagination
  }
  ```

## Implementation Steps

1. **Start with Caching (Highest ROI)**
   - Implement the cache layer first
   - Integrate with existing fetch_projects function

2. **Add Memory Indexing**
   - Build a simple index for client-side search
   - Implement basic filter/sort operations

3. **Implement Progressive Loading**
   - Modify project list component to load in batches
   - Add intersection observer for infinite scroll

## Benefits

1. **Performance:** Reduced API calls and data processing
2. **UX Improvements:** Faster search, filter, and scroll operations
3. **Network Resilience:** Better handling of slow connections
4. **Memory Efficiency:** Only render what's visible

## Future Considerations

- Consider using IndexedDB for persistent cache
- Implement compression for cached data
- Add background sync for offline support

## 7. Lightweight Reputation & Comment System for Bene

## Overview

A lightweight reputation and comment system for Bene platform using Ergo tokens, focused on verified identities and creator-donor communication rather than a full critic network.

## Token Structure

Using Ergo tokens with data in registers:
- **R4**: Type tag (verification, comment, spam-alert)
- **R5**: Reference type (project, comment)
- **R6**: Target ID (project hash, comment token ID)
- **R7**: User wallet address
- **R8**: Positive/negative flag
- **R9**: Content data (JSON)

## Implementation

### Core Components

```typescript
// ReputationProof.ts - Basic token structure definition
export interface ReputationToken {
  type: string;      // verification, comment, spam-alert
  referenceType: string;  // project, comment
  referenceId: string;
  walletAddress: string;
  isPositive: boolean;
  data: object;
}

// Modified generate_reputation_proof.ts from sigma-reputation-panel
export async function mintReputationToken(
  tokenType: string,
  referenceType: string,
  referenceId: string,
  isPositive: boolean,
  data: object
): Promise<string> {
  // Simplified implementation
  // Uses wallet to sign and mint token with correct registers
  return transactionId;
}
```

### Comment System

```typescript
// Post a comment on a project
async function postComment(projectId, content) {
  return mintReputationToken(
    "comment",
    "project",
    projectId,
    true,
    { text: content, timestamp: Date.now() }
  );
}

// Fetch comments for a project
async function getProjectComments(projectId) {
  // Query blockchain for tokens with:
  // R4="comment", R5="project", R6=projectId
  return commentTokens;
}
```

### Verification System

```typescript
// Admin verification approval
async function approveVerification(walletAddress) {
  return mintReputationToken(
    "verification",
    "wallet",
    walletAddress,
    true,
    { timestamp: Date.now() }
  );
}

// Check if user is verified
async function isVerified(walletAddress) {
  // Find verification tokens where R7=walletAddress
  return verificationExists;
}
```

## Integration Points

### Backend

1. **API Endpoints**:
   - `/api/comments/{projectId}` - Get project comments
   - `/api/verification/{address}` - Check verification status

2. **Blockchain Integration**:
   - Adapt `generate_reputation_proof.ts` from sigma-reputation-panel
   - Add register query functions to fetch.ts

### Frontend

1. **Components**:
   - Comment section on project pages
   - Verification badges on user profiles
   - Admin verification dashboard

2. **UI Example** (Comment Section):
   ```svelte
   <!-- Comment.svelte - Simplified -->
   <div class="comments">
     <div class="new-comment">
       <textarea bind:value={comment}></textarea>
       <button on:click={submitComment}>Post</button>
     </div>
     
     {#each comments as comment}
       <div class="comment">
         <span class="address">{comment.walletAddress}</span>
         <p>{comment.data.text}</p>
       </div>
     {/each}
   </div>
   ```

## Cost Management

1. **Optional off-chain storage**: Store only content hashes on-chain
2. **Batch transactions**: Group multiple operations
3. **Fee previews**: Show cost before posting

## Security Features

1. **Wallet signatures**: Ensure comment authenticity
2. **Admin approval flow**: For verification status
3. **Spam detection**: Flag comments with multiple spam reports




## 8. Wallet Balance Controls Enhancement - Implementation Status

## Overview
This point outlines the status of the wallet balance controls enhancement, which aims to improve user experience by dynamically updating UI elements based on available funds and preventing failed transactions.

## Currently Implemented Features

### 1. Balance Variables and Tracking
- ✅ Reactive wallet balance variables in ProjectDetails.svelte:
  ```typescript
  let userErgBalance = 0;
  let userProjectTokenBalance = 0;
  let userTemporalTokenBalance = 0;
  let maxContributeAmount = 0;
  let maxRefundAmount = 0;
  let maxCollectAmount = 0;
  let maxWithdrawTokenAmount = 0;
  let maxWithdrawErgAmount = 0;
  ```

- ✅ Comprehensive `getWalletBalances()` function that:
  - Fetches and calculates current ERG balance
  - Retrieves user's project and temporal token balances
  - Calculates maximum amounts for various actions based on balances
  - Handles project owner-specific balance calculations

- ✅ Balance updating triggered when connection state changes:
  ```typescript
  $: if ($connected) {
      getWalletBalances();
  }
  ```

### 2. Balance Display and Periodic Updates
- ✅ ERG balance display in the navigation bar in App.svelte
- ✅ Periodic wallet info updates (every 30 seconds) in App.svelte:
  ```typescript
  let balanceUpdateInterval = setInterval(updateWalletInfo, 30000);
  ```

### 3. Initial Integration
- ✅ Balance awareness in some form setups with `getWalletBalances()` calls before transaction setup
- ✅ Basic balance variables structure for tracking different token types

## Remaining Implementation Tasks

### 1. Input Field Validation
-  Add proper max value constraints to all transaction input fields based on available balances:
  ```svelte
  <Input
    type="number"
    bind:value={value_submit}
    min="0"
    max={info_type_to_show === "buy" ? maxContributeAmount : 
         info_type_to_show === "" ? maxRefundAmount :
         info_type_to_show === "dev" ? maxWithdrawTokenAmount :
         info_type_to_show === "dev-collect" ? maxWithdrawErgAmount : 0}
  />
  ```

### 2. Button Disabling Logic
-  Add disabled state to action buttons based on available funds:
  ```svelte
  <Button 
    on:click={setupBuy}
    disabled={!$connected || maxContributeAmount <= 0 || project.sold_counter >= project.total_pft_amount}
  >
    Contribute
  </Button>
  ```

-  Ensure all action buttons have appropriate disabling conditions:
  - Contribute button: disabled when maxContributeAmount <= 0
  - Refund button: disabled when maxRefundAmount <= 0
  - Collect button: disabled when maxCollectAmount <= 0
  - Owner actions: disabled based on withdrawal limits

### 3. User Feedback Mechanisms
-  Add helpful visual feedback explaining why buttons might be disabled:
  ```svelte
  {#if $connected && maxContributeAmount <= 0 && project.sold_counter < project.total_pft_amount}
    <div class="insufficient-funds-message">
      Insufficient funds for contribution
    </div>
  {/if}
  ```

-  Implement tooltips or helper text showing maximum available amounts

### 4. Balance Refresh Consistency
-  Ensure all transaction setup functions call `getWalletBalances()` before opening forms:
  ```typescript
  function setupRefund() {
      getWalletBalances(); // Add this call to refresh balances first
      // Rest of the setup function...
  }
  ```

-  Add balance refreshes before each transaction execution

### 5. ProjectCard.svelte Enhancements
-  Add balance awareness to project cards to indicate which projects user can contribute to
-  Visual indicators on cards showing if user has sufficient funds

### 6. Real-Time Updates
-  Implement balance updates on form input changes to show real-time affordability
-  Add reactive updates to max amount when wallet balance changes during form interaction

## Next Steps
1. Focus on implementing the input validation constraints first
2. Add button disabling logic based on available balances
3. Implement user feedback mechanisms and tooltips
4. Ensure consistent balance refreshing across all transaction setups
5. Add visual enhancements to project cards

## Benefits After Completion
- ✓ Prevention of failed transactions due to insufficient funds
- ✓ Clear visual feedback for users about available actions
- ✓ Improved overall user experience with dynamic UI updates
- ✓ Reduced user confusion and frustration




## 9. Project Analytics and Dashboard

**Technical Implementation Approach:**

The analytics dashboard will provide project creators and backers with valuable insights into funding progress, contributor demographics, and project performance metrics.

### Data Collection Strategy

1. **Metrics to Track:**
   - Funding Progress: Percentage funded, rate of contributions
   - Contributor Analysis: Number of unique contributors, return rate
   - Token Distribution: PFT token distribution statistics
   - Time-Based Analysis: Peak contribution times, funding velocity

2. **Data Storage Approach:**
   - On-chain data: Extract core metrics from blockchain events
   - Off-chain aggregation: Cache and process metrics for efficient display
   - Time-series storage: Track changes over time for trend analysis

### Files to Modify:

1. **Backend Data Processing:**
   - `src/lib/ergo/fetch.ts` - Add methods to query historical box data
   - `src/lib/common/analytics.ts` (new): Create analytics processing functions
   - `src/lib/ergo/platform.ts` - Add analytics retrieval methods

2. **Frontend Components:**
   - `src/routes/ProjectAnalytics.svelte` (new): Create analytics dashboard view
   - `src/routes/ProjectDetails.svelte` - Add analytics preview section
   - `src/lib/components/charts/` (new directory): Add visualization components

3. **Data Storage:**
   - `src/lib/common/store.ts` - Add analytics data stores
   - `src/lib/common/persistence.ts` (new): Implement local caching for analytics

### Implementation Process:

1. **Data Collection:**
   ```
   // Pseudocode for contribution analysis
   function analyzeContributions(projectId) {
     const contributionEvents = await getProjectEvents(projectId, 'contribution');
     
     return {
       totalContributors: new Set(contributionEvents.map(e => e.address)).size,
       contributionsByTime: groupAndCountByTimeframe(contributionEvents),
       averageContribution: calculateAverage(contributionEvents.map(e => e.amount)),
       // Additional metrics
     };
   }
   ```

2. **Visualization Framework:**
   - Implement responsive chart components using lightweight libraries 
   - Create dashboard layout with filterable time ranges
   - Design printable/exportable reports

3. **Integration with Existing UI:**
   - Add analytics tab to project details page
   - Create dedicated analytics dashboard for project owners
   - Add summary metrics to project cards

This analytics system will enable:
1. Project owners to understand funding patterns and optimize their campaigns
2. Contributors to evaluate project momentum and community engagement
3. Platform administrators to identify successful project patterns

## 10. UI/UX Improvements

To enhance the user experience and make the platform more accessible and intuitive, I'll implement several key UI/UX improvements:

### 1. File Upload System for Project Media

**Current Issue:** Users must provide URLs for project images, which is cumbersome and error-prone.

**Proposed Solution:**
- Replace URL input with direct file upload interface
- Implement client-side image optimization
- Add image preview functionality
- Support multiple image uploads for project galleries

**Implementation Approach:**
- Modify `NewProject.svelte` to include file upload components
- Create a media storage solution using IPFS or similar decentralized storage
- Implement file type validation and size restrictions

### 2. Enhanced Content Creation

**Current Issue:** Text-only description input is limiting for project creators.

**Proposed Solution:**
- Implement a rich text editor for project descriptions
- Add AI-powered description summarization
- Create templates for common project types
- Add keyword suggestion tools for better discoverability

**Implementation Files:**
- Create new component: `src/lib/components/RichTextEditor.svelte`
- Add AI integration module: `src/lib/common/ai-summary.ts`
- Modify `NewProject.svelte` to use enhanced content components

### 3. Improved Navigation and Layout

**Current Issue:** Infinite scrolling creates navigation problems and resource issues.

**Proposed Solution:**
- Replace infinite scroll with efficient pagination (as detailed in section 2.4)
- Implement a proper site footer with important links and information
- Add breadcrumb navigation for improved orientation
- Create a consistent layout system with better responsive behavior

**Key Components:**
- New `Footer.svelte` component with platform information and links
- Enhanced navigation with clear hierarchy and breadcrumbs
- Improved loading states and transitions between pages

### 4. Accessibility Enhancements

**Current Issue:** Platform has limited accessibility features.

**Proposed Solution:**
- Implement proper ARIA attributes throughout the application
- Add keyboard navigation support
- Ensure color contrast meets WCAG standards
- Support screen readers with appropriate text alternatives

These UI/UX improvements will significantly enhance the platform's usability, making it more accessible to a wider audience while simplifying the project creation process for fundraisers.

## 11. Bountiful: Single-Judge Bounty Platform for Ergo

### Core Concept

Bountiful is a streamlined bounty platform on Ergo that enables funding development tasks through open competition, where rewards are only released when a trusted judge approves a solution.

## Mathematical Model & Logic

### Reward Calculation

The reward amount (R) grows over time with additional contributions:

R = Σ(C_i) for all contributions i

Where:
- C_i represents individual contributions
- The total reward is the sum of all contributions

### Dev Fee Formula

For a successful solution, the dev fee is calculated as:

DevFee = R × (BPS / 10000)

Where:
- R is the total reward amount
- BPS is the basis points (100 = 1%)

### Remaining Reward

The solution provider receives:

Solution_Reward = R - DevFee

### Refund Mechanism (if deadline expires)

For each contributor:

Refund_i = C_i

### Timelock Logic

Funds are locked until one of two conditions is met:
1. Judge_Approval = TRUE, or
2. Current_Height > Deadline_Height

### Logical Contract Flow

1. Bounty Contract Creation:
   - Input: Initial_Funds, Judge_PubKey, Description_Hash, Creator_PubKey, Deadline, DevFee_BPS
   - Output: Locked bounty contract with registers R4-R8 storing parameters

2. Contribution Logic:
   - When(New_Contribution):
     - Bounty_Value += Contribution_Amount

3. Approval Logic:
   - IF(Judge_Signature_Valid AND HEIGHT <= Deadline):
     - Creator_Reward = Bounty_Value × (DevFee_BPS/10000)
     - Developer_Reward = Bounty_Value - Creator_Reward
     - RETURN TRUE
   - ELSE:
     - RETURN FALSE

4. Refund Logic:
   - IF(HEIGHT > Deadline):
     - Enable proportional refunds to contributors
     - RETURN TRUE
   - ELSE:
     - RETURN FALSE

## Core Components

### 1. Bounty Contract State

The contract maintains a state defined by:
- R4: Description hash (IPFS hash of task description)
- R5: Creator's address (for receiving dev fee)
- R6: Judge's public key
- R7: Deadline height
- R8: Dev fee percentage (in basis points)

### 2. Logical Flows

#### Bounty Creation
1. Creator defines: Task, Reward, Judge, Deadline, Fee %
2. Contract deploys with initial funds locked

#### Funding
- Contributors can add funds anytime before a solution is approved
- R(t) = R(0) + Σ(contributions) as a function of time

#### Verification
- Judge reviews submissions off-chain
- Signs transaction if solution meets requirements
- Verification rule: Judge_PK && (Dev_Fee_Box && Solution_Box)

#### Time-Based Safeguard
- If HEIGHT > Deadline && No_Approved_Solution:
  - Enable refund claims

## Security Logic

1. Judge authorization: Single signature verification
2. Reward distribution: Deterministic split based on DevFee_BPS
3. Deadline enforcement: Block height comparison

## Integration with Bene

Bountiful extends Bene's core wallet infrastructure and token handling, while introducing new validation logic and incentive structures.

## Conclusion

This model provides a pragmatic balance between trustlessness and efficiency, enabling incentivized development through clearly defined mathematical rules for reward distribution and verification.

## References

1. [Ergo Platform Documentation](https://docs.ergoplatform.com/) - Official documentation for Ergo blockchain development
2. [ErgoScript Language Specification](https://github.com/ScorexFoundation/sigmastate-interpreter/blob/develop/docs/LangSpec.md) - Technical reference for writing Ergo smart contracts
3. [Ergo AppKit](https://github.com/ergoplatform/ergo-appkit) - JVM toolkit for Ergo blockchain integration
4. [Ergo Node API](https://github.com/ergoplatform/ergo/blob/master/src/main/resources/api/openapi.yaml) - API for Ergo node interaction
5. [Nautilus JS API](https://github.com/nautiluscs/nautilus-js) - JavaScript library for Nautilus wallet integration
6. [IPFS HTTP Client](https://github.com/ipfs/js-ipfs/tree/master/packages/ipfs-http-client) - JavaScript client library for IPFS
7. [Svelte](https://svelte.dev/) - UI framework for efficient component-based development
8. [Ergo Explorer API](https://github.com/ergoplatform/explorer-backend) - API for querying blockchain data
10. [SigmaUSD](https://github.com/anon-real/sigma-usd) - Reference for handling stablecoins on Ergo





## 12. Smart Contract Architecture

To support these new features, I'll extend the current contract architecture:

```markdown
// Extended contract hierarchy
BeneBaseContract (base functionality)
├── FundraisingContracts
│   ├── ERGFundraisingContract (existing)
│   ├── MultiTokenFundraisingContract (new)
│   │   ├── StablecoinHandler (SigUSD integration)
│   │   ├── TokenRegistry (supported tokens list)
│   │   └── OracleDataConsumer (price feeds)
│   └── MilestoneFundraisingContract (new)
│       ├── MilestoneTracker
│       └── VotingMechanism
├── BountifulContracts (new)
│   ├── SingleJudgeBountyContract (primary implementation)
│   │   ├── JudgeVerifier
│   │   ├── DeadlineEnforcer 
│   │   └── RewardDistributor
│   └── BountyIndexer (registry of active bounties)
└── SupportContracts
    ├── ReferralSystem (with anti-exploitation mechanism)
    ├── AnalyticsDataCollector
    ├── MultiWalletConnector
    └── PaginationHandler (for optimized data fetching)
```


## 13. Detailed Timeline

#### Community Bonding Period (May 8 - June 1)
- Study existing Bene Ergo codebase thoroughly
- Research Ergo token standards and stablecoin implementations
- Set up ErgoScript development and testing environment
- Create detailed architecture documents
- Collaborate with mentors on priority setting

#### Week 1-2 (June 2 - June 15): Multi-Token Support Core
- Modify existing contract to handle arbitrary tokens
- Implement token registry system
- Create basic valuation mechanism
- Develop core tests

#### Week 3-4 (June 16 - June 29): Stablecoin Integration
- Implement SigUSD-specific integration
- Create token selection UI components
- Develop exchange rate oracle integration
- Build comprehensive test suite

#### Week 5-6 (June 30 - July 13): Multi-Wallet Integration
- Develop wallet adapter interface
- Implement specific wallet integrations (Nautilus, Safew)
- Create wallet connection UI components
- Test with various wallet providers

#### Mid-term Evaluation (July 14 - July 18)
- Complete documentation for implemented features
- Submit mid-term report
- Gather feedback from mentors and community

#### Week 7-8 (July 14 - July 27): Bountiful Smart Contract Implementation
- Design and implement SingleJudgeBountyContract
- Create judge verification mechanism
- Implement deadline enforcement
- Develop reward distribution logic
- Build bounty creation and management UI

#### Week 9-10 (July 28 - August 10): Project Analytics & Milestone-Based Funding
- Implement project analytics dashboard
- Create data collection mechanisms
- Develop milestone tracking system
- Implement voting mechanism for milestone approval
- Build milestone management UI components

#### Week 11 (August 11 - August 17): Referral System & Enhanced UX
- Implement referral tracking mechanism
- Create anti-exploitation safeguards
- Develop enhanced wallet balance controls
- Build pagination system for project listings
- Implement rich text editor for descriptions

#### Week 12 (August 18 - August 24): Testing & Documentation
- Comprehensive testing of all components
- Security review and fixes
- Complete user and developer documentation
- Prepare demonstration materials

#### Final Week (August 25 - September 1): Buffer & Final Submission
- Buffer time for unexpected challenges
- Final polishing and optimization
- Code cleanup and performance optimizations
- Final submission preparation
- Knowledge transfer to project maintainers

##14. Technical Challenges and Solutions


#### Challenge 1: Referral System Implementation
Creating a secure referral system requires preventing exploitation while properly tracking contributions.

**Solution:** Design a referral tracking mechanism with anti-exploitation safeguards:
```ergoscript
// Referral system implementation
{
  // Referral data structure in R9
  val referralData = {
    // Format: [(referrer_address, referral_percentage, min_contribution)]
    SELF.R9[Coll[(Coll[Byte], Int, Long)]].get
  }
  
  // Validate referral reward distribution
  val isValidReferralReward = {
    // Extract contribution details
    val contribution = INPUTS(0).value
    val referrerAddress = INPUTS(0).R8[Coll[Byte]].get
    
    // Find referral data for this referrer
    val referralInfo = referralData.filter(data => data._1 == referrerAddress)
    
    if (referralInfo.size > 0) {
      val percentage = referralInfo(0)._2
      val minContribution = referralInfo(0)._3
      
      // Check minimum contribution requirement
      val meetsMinimum = contribution >= minContribution
      
      // Calculate reward
      val reward = (contribution * percentage) / 100
      
      // Verify reward box
      val hasReferralOutput = OUTPUTS.size >= 3
      val referralOutput = OUTPUTS(2)
      val correctReferralAddress = referralOutput.propositionBytes == referrerAddress
      val correctReferralAmount = referralOutput.value >= reward
      
      meetsMinimum && hasReferralOutput && correctReferralAddress && correctReferralAmount
    } else {
      // No referral data found
      true
    }
  }
  
  // Anti-exploitation safeguards
  val isNotExploiting = {
    // Prevent self-referrals
    val notSelfReferral = INPUTS(0).propositionBytes != INPUTS(0).R8[Coll[Byte]].get
    
    // Limit referral rewards to reasonable percentage
    val reasonablePercentage = referralData.forall(data => data._2 <= MAX_REFERRAL_PERCENTAGE)
    
    notSelfReferral && reasonablePercentage
  }
  
  // Include referral validation in main contract
  isValidReferralReward && isNotExploiting
}
```

## 15. Why This Matters: Impact on Ergo Ecosystem

The proposed enhancements will significantly impact the Ergo ecosystem:

1. **Increased Funding Options**: Multi-token support enables projects to accept various assets beyond ERG
2. **Improved User Experience**: Enhanced wallet integrations and balance controls prevent failed transactions
3. **Enhanced Accountability**: Milestone-based funding provides greater security for contributors
4. **Ecosystem Growth**: Referral systems incentivize marketing and community expansion
5. **Data-Driven Projects**: Analytics dashboards provide creators with actionable insights
6. **Cross-Ecosystem Alignment**: Feature parity with EVM creates a unified experience for users

## 16. Why Me?

I am uniquely qualified to enhance the Benefaction Platform due to my combination of technical skills and prior contributions to the project:

#### Prior Contributions to Benefaction Platform
- Designed and developed the entire landing page for Benefaction Platform (https://github.com/StabilityNexus/Bene-LandingPage)
- Implemented significant UI improvements to the Ergo version through multiple pull requests:
  - [PR #31](https://github.com/StabilityNexus/BenefactionPlatform-Ergo/pull/31):
    - Fixed Bene logo text alignment for better visual balance
    - Implemented the Navbar approved by Bruno and Josemi 
    - Implemented hamburger menu for navigation on mobile screens
    - Improved vertical scrollbar in Fundraising Projects section
    - Enhanced Loading Projects indicator with skeleton loader animation
  
  - [PR #33](https://github.com/StabilityNexus/BenefactionPlatform-Ergo/pull/33):
    - Fixed scrollbar disappearance issue when dropdown menus appear in dark mode
    - Refined design and layout of project cards to improve visual appeal
    - Eliminated dual scrollbars when displaying project details
    - Implemented the New UI for ProjectDetails.tsx Page 
    - Implemented single page layout for better user experience
    - Added search functionality to help users find projects more easily

  - Added responsive design elements and improved mobile navigation
  - Enhanced visual consistency across the platform
- Actively collaborated with the team through Discord to identify and prioritize enhancement opportunities

#### Technical Skills
- **Frontend Development**: Strong proficiency in Svelte, TypeScript, and modern CSS frameworks
- **Blockchain Experience**: Familiar with UTXO-based systems and Ergo blockchain architecture
- **Smart Contract Knowledge**: Understanding of ErgoScript fundamentals for effective UI/contract integration
- **UI/UX Design**: Skilled in creating intuitive, responsive interfaces that enhance user experience

#### Specific UI Improvements Implemented
- Fixed scrollbar disappearance issue in dark mode when dropdowns appear
- Enhanced card design and layout for improved visual appeal and user engagement
- Eliminated dual scrollbars when displaying project details
- Implemented search functionality to help users find projects as the platform scales
- Aligned Bene logo text properly with the icon for better visual balance
- Created responsive hamburger menu for mobile navigation
- Improved loading indicators with modern skeleton loaders and animations

#### Research Experience at IIT Delhi
During my 6-month internship at IIT Delhi, I contributed to several research projects:
- Developed an end-to-end dashboard for the Hero project integrating EEG headsets to monitor safety signals in two-wheeler helmet systems
- Built interview analysis software with sentiment analysis and comprehensive metrics visualization
- Created a standalone application replicating the functionality of Taguette, an open-source qualitative research tool

#### Professional Experience
I've worked with two startups in frontend development roles where I:
- Designed and implemented complete CRM dashboard systems
- Developed production-ready user interfaces for complex business applications
- Applied modern web development practices in real-world business contexts

#### Achievements
- Won two hackathons (organized by kimo.ai and my college)
- Published technical articles on GeeksforGeeks
- All experience certificates available on my LinkedIn profile: https://www.linkedin.com/in/aditya-gupta-732857233/

### 17. References

1. Ergo Platform Documentation: [https://docs.ergoplatform.com/](https://docs.ergoplatform.com/)
2. SigUSD Protocol: [https://sigmausd.io/](https://sigmausd.io/)
3. ErgoDEX Documentation: [https://docs.ergodex.io/](https://docs.ergodex.io/)
