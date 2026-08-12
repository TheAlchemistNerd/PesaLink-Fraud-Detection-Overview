# Fraud Transactions, Graph Topologies, and Insider Typologies

This document synthesizes the structural topological signatures of fraud within the PesaLink interbank network. It combines explicit multi-hop Graph Neural Network (GNN) architectures with the underlying business, economic, and demographic drivers that dictate transaction behaviours across the Kenyan financial landscape.

---

## 1. Technical Context: GNN Topological Graph Signatures

Traditional rule-based systems analyze transactions in isolation. Conversely, a Graph Neural Network (GNN) evaluates the *topology* of the financial ecosystem, mapping relationships as nodes (accounts/devices) and edges (transactions).

### The Mathematical Baseline: Component Asymmetry
Genuine banking transactions are heavily directed and almost always acyclic. Wealth flows unidirectionally down an economic cascade (Employer $\to$ Employee $\to$ Supermarket $\to$ Distributor $\to$ Manufacturer $\to$ Corporate Tax). Loops are naturally rare. When closed loops *do* occur rapidly, the GATConv (Graph Attention Network) layer instantly flags them as highly suspicious round-tripping or wash-trading.

### Bipartite Subgraph Structures (The Benign Hub)
In a scale-free graph, certain nodes naturally achieve massive in-degree metrics. For example, a consumer-to-merchant bipartite subgraph ($U \times M$) features individual accounts ($U$) continuously interacting with a utility merchant ($M$), such as Kenya Power. 
- **Graph Signature**: The high in-degree of the merchant node and the temporal regularity of recurring edges (e.g., monthly bill payments bound to the same IMSI) make this a near-zero-risk pattern in the GNN's embedding space.

### Active Topology Fault Isolation
The network is structured around a decentralized peer-to-peer (P2P) ring topology. If a bank node drops offline (e.g., due to a targeted breach or heartbeat timeout), the Active Topology Manager detects the failure within 2.0 seconds and dynamically reconfigures the communication path to bypass the dead node, closing the loop to ensure continuous zero-trust federated learning.

---

## 2. Demographic Context: Homophily vs. Heterophily

Fraud often manifests when demographic realities fail to match transactional topologies. The GNN relies heavily on measuring node similarity and clustering behaviors to determine risk.

### Localized Homophily
Accounts naturally exhibit behavioral alignment based on demographic or regional grouping (homophily). 
- **Example**: A university student account primarily interacts with other student accounts or low-value food merchants. This dense localized clustering is a safe, expected topological pattern.

### Structural Heterophily
When an established pattern breaks, it signals an anomaly. 
- **Example**: A sudden, high-velocity transaction from a low-KYC student demographic directly into a high-volume, corporate-tier escrow account represents intense structural heterophily, immediately triggering an anomaly flag in the graph embedding space.

### Bipartite Homophily Analysis
GNN models measure whether nodes with low KYC tiers (or histories of SIM swaps) are disproportionately transacting with highly trusted internal bank pool accounts or specific "whitelisted" merchant hubs, highlighting potential inside exposure points where demographics and transaction limits do not logically align.

---

## 3. Business & Economic Context: The Cost of Topology

### Mule Account Syndicates
From a demographic and economic perspective, fraud syndicates leverage networks of dormant or fraudulently-opened accounts (often exploiting vulnerable, low-income demographics for ID registration) to receive illicit funds.
- **Graph Signature**: These nodes exhibit a distinctive **in-degree spike followed by rapid out-degree** in temporal graph analysis: high in-flow velocity, followed by a single terminal withdrawal edge at an agent node or ATM.

### The Weaponization of the "White-Listed Hub"
Legitimate merchants and payment switches are high-degree "hubs" because thousands of users transact with them. However, insiders actively weaponize this economic reality. Insiders with database privileges manually alter the feature characteristics of a specific fraud ring account, marking it as an "Exempt Merchant". This allows fraud rings executing SIM-swaps to route massive volumes of stolen money into this single hub account without triggering basic transactional volume blocklists.

---

## 4. Specific Insider Fraud Graph Scenarios (Verbatim Extraction)

Insider fraud leaves profound topological scars on the transaction graph itself, regardless of whether administrative system logs were wiped. The following scenarios explicitly model these topologies:

### 4.1 The Closed Clearing Loop (The Cash-Out Cycle)
Insider fraud often involves an employee within a bank or a mobile wallet operator (like M-Pesa) manipulating clearing ledgers or overriding system credit limits to manufacture artificial funds. 
- **The Instance**: A corrupt bank employee creates or reactivates a mule account with a fake or bypassed KYC profile. They temporarily exploit an internal vault clearing window to execute an unbacked transaction to an external account. To hide the missing cash from automated end-of-day ledger reconciliations, the funds are rapidly pushed through multiple hops across different banks (via PesaLink) before circling back to settle the original internal account right before the audit window closes. 
- **The Graph Signature**:
  - *The Directed Cycle*: Your GNN flags a strict directed cycle occurring within a tight temporal window (e.g., under 15 minutes).
  - *Edge Weight Invariance*: The transaction amount remains identical or drops by precise percentages (reflecting transaction fees or processing cuts) across the entire multi-hop path.

### 4.2 High-Degree Node Bypasses (The White-Listed Hub)
- **The Instance**: An insider with database or application administration privileges manually alters the feature characteristics of a specific fraud ring account, explicitly marking it as an "Exempt Merchant" or "System Trusted Account." This allows a fraud ring executing SIM-swaps or card-cloning scams to route massive volumes of stolen money into this single hub account without triggering basic transactional volume blocklists.
- **The Graph Signature**: A high-degree node whose structural connections are heavily clustered with recently registered wallets, accounts with high SIM-swap frequencies, or low-KYC profiles, throwing off an intense anomaly signature to the GATConv attention heads (Bipartite Mixing Anomaly).

### 4.3 Layered Dispersal and Instant Re-Aggregation (The Star-Network)
- **The Instance**: An insider triggers an unauthorized bulk payment or a manual adjustment from an internal ledger pool. The money is distributed simultaneously to 50 low-tier mobile wallets. Within minutes, those 50 wallets push the funds across 3 or 4 extra hops, which eventually converge into a single merchant account or a cryptocurrency gateway cash-out point in Nairobi. 
- **The Graph Signature**:
  - *The Hourglass Structure*: The graph transitions from a single internal source node, splits into a wide parallel array of unique intermediate nodes, and then instantly contracts back into a single terminal sink node.
  - *Structural Coherence*: While individual paths appear acyclic and disconnected to basic rule engines, the GNN's attention heads compute high structural coherence across the parallel paths, indicating automated, coordinated execution.
