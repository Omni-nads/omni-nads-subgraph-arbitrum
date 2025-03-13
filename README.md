# OmniNads Subgraph

This subgraph indexes the **OmniNadsConsumer** contract, capturing events (such as `Transfer`, `BaseURISet`, and bridging events like `ONFTReceived`/`ONFTSent`) to build an off-chain view of token ownership, state changes, and URI updates.

## Table of Contents

1. [Overview](#overview)  
2. [Architecture & Design Choices](#architecture--design-choices)  
3. [Technical Implementation](#technical-implementation)  
4. [Requirements](#requirements)  
5. [Installation & Setup](#installation--setup)  
6. [Local Development](#local-development)  
7. [Deployment](#deployment)  
8. [Usage](#usage)  

---

## Overview

- **Purpose**: This subgraph listens to events from the **OmniNadsConsumer** smart contract on Arbitrum (or any other target chain), storing the resulting data in a queryable format. Users can query a list of tokens, their current `owner`, `tokenURI`, and other relevant metadata.  
- **Features**:  
  - Tracks minted NFTs (via `Transfer` from zero).  
  - Handles bridging events (`ONFTReceived` / `ONFTSent`) by decoding combined token IDs.  
  - Dynamically updates `tokenURI` when the base URI changes (`BaseURISet`).  
  - Removes tokens from the subgraph if their `owner` is the zero address.  

---

## Architecture & Design Choices

1. **Entities**:  
   - **Token**: Represents each NFT, storing fields like `owner`, `tokenId`, `tokenURI`, `tokenState`, and relevant block/time details.  
   - **Global**: A singleton entity (id = `"global"`) that tracks the current `baseURI` and an array of `tokenIds`. Used to batch-update tokens when `BaseURISet` fires.  
   - **Immutable Event Entities** (like `Transfer`, `ONFTReceived`, etc.): Each event is stored as a unique entity for historical tracking.  

2. **Design Considerations**:  
   - **Bridging** is handled by decoding the token ID and state from a single `BigInt`. When an event implies the token is leaving (owner=0), the subgraph removes that token from active indexing.  
   - **Base URI Updates** can change existing tokens’ URIs. The subgraph re-derives these URIs by iterating through all minted token IDs in the `Global` entity.  
   - **Non-Nullable Fields**: `Token.owner` is declared as `Bytes!` in the schema, so the code ensures it is always set. If the event does not provide an owner, the subgraph calls the contract with `ownerOf(tokenId)` or defaults to `0x0`.  

---

## Technical Implementation

- **Language**: AssemblyScript (in `src/omni-nads-consumer.ts`).  
- **Event Mappings**: The subgraph listens to events declared in [`subgraph.yaml`](./subgraph.yaml). Each event references a handler function (e.g. `handleTransfer`, `handleBaseURISet`, etc.), which updates the relevant entities in the store.  
- **Contract Decoding**: We rely on the `OmniNadsConsumer` ABI to decode event parameters (like `tokenState`), plus a helper function `decodeTokenInfo(encodedTokenId)` for bridging.  
- **Owner-of Calls**: If an event doesn’t provide the new owner, the subgraph tries `contract.try_ownerOf(tokenId)`. If that reverts, a zero‐address fallback is used. This ensures non‐nullable fields are never missing.  

---

## Requirements

1. **Node.js** (v14 or higher recommended)  
2. **Yarn** or **npm** (for installing dependencies)  
3. **@graphprotocol/graph-cli**:  
   - Used for running `graph codegen`, `graph build`, and `graph deploy`.  
4. **Contracts & ABIs**:  
   - The subgraph references the `OmniNadsConsumer.json` ABI to decode events.  
5. **Git / Version Control**:  
   - Typically a Git repository to manage changes.  

> **External Dependencies**:  
> - The Graph CLI  
> - The [Arbitrum network config](https://chainlist.org/) or whichever chain you’re targeting  
> - Any environment variables if you plan to do a `graph deploy` to a hosted service or to a decentralized Graph node.

---

## Installation & Setup

1. **Clone** this repository:
   ```bash
   git clone https://github.com/<your-org>/<repo-name>.git
   cd <repo-name>
   ```

2. **Install Dependencies**:
   ```bash
   yarn install
   ```
   or
   ```bash
   npm install
   ```

3. **Review the Subgraph Configuration**:  
   - Open [`subgraph.yaml`](./subgraph.yaml) to confirm contract addresses, ABIs, event handlers, and data sources are correct for your environment.

4. **(Optional) Set Environment Variables**:  
   - If you plan to deploy to a hosted or decentralized Graph node, you may need `GRAPH_ACCESS_TOKEN` or similar.  
   - For local development, environment variables are typically unnecessary unless using a custom endpoint.

---

## Local Development

1. **Generate AssemblyScript Types**:
   ```bash
   yarn run codegen
   ```
   This uses the contract ABIs and your schema (`schema.graphql`) to generate TypeScript bindings in `generated/`.

2. **Build the Subgraph**:
   ```bash
   yarn run build
   ```
   This will compile the AssemblyScript mappings (in `src/`) to WebAssembly, and validate your GraphQL schema.

3. **Run Tests (Optional)**:
   - If you have custom test scripts or a local Graph Node environment, you can run them here.  

---

## Deployment

You can deploy to:

- **The Graph Hosted Service**  
- **Decentralized Graph Network**  
- **Local Graph Node** instance

### Deploying to Hosted Service

1. **Authenticate**:
   ```bash
   graph auth --product hosted-service <ACCESS_TOKEN>
   ```
2. **Deploy**:
   ```bash
   graph deploy \
     --product hosted-service \
     <GITHUB_USER_OR_ORG>/<SUBGRAPH_NAME>
   ```

### Deploying to Decentralized Network

1. **Authenticate** with your CLI wallet.  
2. **Update** `subgraph.yaml` with the network config and address.  
3. **Deploy**:
   ```bash
   graph deploy \
     --studio <YOUR_SUBGRAPH_SLUG>
   ```

### Deploying Locally
1. Launch a local Graph Node + IPFS + Postgres environment.  
2. Configure the local node in `subgraph.yaml`.  
3. Deploy:
   ```bash
   graph create <LOCAL_SUBGRAPH_NAME>
   graph deploy <LOCAL_SUBGRAPH_NAME>
   ```

---

## Usage

Once deployed, you can run queries against the subgraph endpoint in The Graph Explorer or your local Graph node. Example queries:

```graphql
{
  tokens(first: 5) {
    tokenId
    owner
    tokenURI
    tokenState
  }
}
```

```graphql
{
  onftReceiveds(first: 5, orderBy: blockTimestamp, orderDirection: desc) {
    id
    toAddress
    tokenId
    blockTimestamp
  }
}
```

These queries return real‐time, indexed data from your subgraph, reflecting the state changes of OmniNadsConsumer.

---

**Enjoy building with The Graph!**  
Feel free to submit issues or pull requests if you find bugs or want additional features.