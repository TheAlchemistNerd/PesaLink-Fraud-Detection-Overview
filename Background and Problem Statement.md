# Background and Problem Statement
## Privacy-Preserving Federated Fraud Detection on Kenya's Interbank Payment Network

---

## 1. Introduction

Kenya's financial technology landscape occupies a singular position in global payments history. A nation that pioneered mobile money with M-Pesa in 2007 has, over the subsequent two decades, assembled one of the most sophisticated real-time payment ecosystems on the African continent. At the centre of the interbank dimension of this ecosystem sits **PesaLink**, operated by Integrated Payment Services Limited (IPSL), a wholly-owned subsidiary of the Kenya Bankers Association (KBA), connecting over 80 financial institutions including commercial banks, microfinance institutions, savings and credit co-operative organisations (SACCOs), and licensed fintech payment service providers (PSPs) [1].

The PesaLink ecosystem, much like global financial networks, operates as a massive, dynamically evolving **scale-free, bipartite graph**. As demonstrated empirically by Saxena et al. (2021), banking transaction datasets naturally form highly skewed power-law degree distributions, where a vast majority of nodes (users) have very few connections, while a minority of 'hub' nodes (utility companies, payment aggregators) possess millions [25]. Furthermore, treating this data purely as a tabular ledger obscures its relational risk. By modeling interoperable payment data as bipartite graphs, institutions drastically improve loan screening and risk monitoring capabilities compared to isolated credit bureau metrics (Rishabh, 2026) [26].

However, this structural topology presents a severe challenge: **Graph Sparsity**. The vast majority of possible connections between accounts are non-existent. Traditional fraud detection cannot accurately correlate sparse data points. Graph-based link-prediction mitigates this by capturing indirect, higher-order pathways, allowing the detection of hidden, multi-hop money laundering rings even when explicit, direct transaction histories between suspicious entities are deliberately obfuscated (Wang et al., 2020) [27].

The maturity of this infrastructure is both its greatest strength and its most acute vulnerability. Real-time, 24/7 settlement removes the historical delay-buffer that compliance teams relied on and compresses the fraud detection window to near-zero. Each transaction must be scored, flagged, or cleared in the time it takes a user to receive an SMS confirmation, typically under two seconds. Fraudsters who previously had hours to move stolen funds across the banking system now operate in milliseconds.

Against this backdrop, Kenya's banking regulators and courts have begun issuing increasingly urgent signals: the sector's current fraud detection architecture is structurally insufficient. The Central Bank of Kenya (CBK) **Bank Supervision Annual Report 2024** documented a **+130.72% year-on-year surge in reported fraud cases** and a **+264.08% explosion in net financial losses**, rising from KES 412 million to KES 1.50 billion in a single reporting period [2]. Simultaneously, the **High Court of Kenya at Machakos** issued a landmark ruling holding that a correct Personal Identification Number (PIN) is not, in itself, a sufficient defence for financial institutions against fraud liability; it ordered Safaricom to bear 60% and Diamond Trust Bank (DTB) 40% of KES 4.42 million stolen from a SIM-swap victim, specifically because the velocity and pattern of transactions should have triggered automated detection [3].

This document provides the research background and formal problem statement for the development of a **privacy-preserving, federated, graph-based fraud detection engine** for the PesaLink network, an architecture that satisfies simultaneously the CBK's operational risk mandates, the ODPC's data minimisation requirements under the Kenya Data Protection Act 2019, and the technical latency demands of a high-velocity real-time settlement rail.

---

## 2. Kenya's Interbank Payment Infrastructure

### 2.1 The National Payments System: Architecture and Hierarchy

The Central Bank of Kenya (CBK) classifies Kenya's National Payment System (NPS) into two structural tiers based on throughput values and volumes [4]:

- **Large Value (Wholesale) Systems**: The **Kenya Electronic Payment and Settlement System (KEPSS)**, a Real-Time Gross Settlement (RTGS) system implemented on 29 July 2005, handles high-value interbank transfers settled individually and continuously in central bank reserve accounts. KEPSS is classified as a Systemically Important Payment System (SIPS) due to its macroeconomic criticality. Regional extensions include the East African Payment System (EAPS) for EAC cross-border flows and the Regional Payment and Settlement System (REPSS) for COMESA flows, both integrated into KEPSS [4].

- **Low Value (Retail) Systems**: This tier includes the Automated Clearing House (ACH) for cheques and Electronic Funds Transfers (EFTs), payment card networks, mobile payment platforms, and critically, **PesaLink**, the instant interbank retail rail.

PesaLink occupies a structurally unique position: a retail system by transaction size (KES 10 to KES 999,999 per transaction) that operates at wholesale clearing speeds. Unlike legacy EFT processing, which involves overnight batch cycles, PesaLink routes funds between banks in real time, with final settlement occurring twice daily through KEPSS net positions [5].

### 2.2 PesaLink: Operational Specifications and Access Channels

PesaLink's central processing hub, operated by IPSL, functions as a **digital clearing switch**: an intermediary routing engine that receives transaction instructions from originating banks, validates counterparty account identifiers, and confirms fund availability before committing the credit to the recipient bank's ledger. No funds transit through IPSL's own balance sheet; the hub is purely an instruction router and settlement netting engine [1, 5].

### 2.3 Institutional Stratification (The 3-Tier Paradigm)
The PesaLink network consists of over 80 participating financial institutions, but volume and capital are highly asymmetrical. To mathematically model and deploy a viable hardware architecture, the ecosystem is stratified into three distinct tiers:
*   **Tier-1 (Equity Anchors)**: 9 large commercial banks controlling >75% of transaction volume. These institutions host the heavy bare-metal GPU/AMX compute clusters and act as federated learning aggregation anchors.
*   **Tier-2 (Premium Subscribers)**: Mid-cap commercial banks and regional microfinance institutions. These entities utilize lighter CPU-bound inference (Intel AMX) and subscribe to the global model weights via a SaaS model.
*   **Tier-3 (Pay-Per-Query consumers)**: Small SACCOs and fintechs with highly erratic volumes, operating via serverless API integrations into the central switch.

**Access channels** through which transactions enter the PesaLink network:

| Channel | Description | Technical Interface |
|---|---|---|
| **Mobile Banking Applications** | Native iOS and Android apps deployed by member institutions | REST/gRPC APIs over TLS 1.3 |
| **USSD** | Session-driven menu interfaces via telco short codes (e.g., \*699#, \*522#) | SS7 / Unstructured Supplementary Service Data protocol |
| **Internet Banking Portals** | Browser-based corporate and retail web applications | HTTPS web sessions |
| **ATMs** | Physical card-present and cardless terminal endpoints | ISO 8583 message format over private VPN tunnels |
| **Agent Banking** | Franchised human-mediated banking points | Aggregator APIs connecting to core banking systems |
| **POS Terminals** | Merchant acquirer hardware for direct-to-account settlement | ISO 8583 / EMV over private network |
| **Open Banking APIs** | Third-party fintech developer access for real-time checkout | OAuth 2.0 + REST |

Each channel produces a distinct **digital fingerprint** for each transaction. The USSD session generates a telco-assigned session token and a device IMSI number; mobile banking produces a device IMEI and IP address; ATM transactions embed a terminal identifier and card number. These multi-modal fingerprints form the raw features of any machine-learning-based fraud detection model applied to the network.

**Transaction types** supported:

- **Send to Account (P2P / P2B)**: Transfers to any participating bank account using the recipient's bank name and account number.
- **Send to Phone**: Transfers using only the recipient's registered mobile phone number, which is linked to a specific bank account via an alias registry maintained by IPSL.
- **Bulk Payments (B2P)**: Enterprise-grade disbursements to multiple bank accounts across different institutions, used for payroll, supplier payments, and government cash transfers.

---

## 3. Transaction Structures: Business Description and Graph-Theoretic Representation

### 3.1 Business Description of a PesaLink Transaction

In business terms, a PesaLink transaction is an **account-to-account (A2A) credit push instruction**. Unlike card debit pull transactions (where a merchant initiates the debit), PesaLink transactions are always initiated by the **payer**: the sending account holder explicitly authorising a credit to a named recipient. This places the fraud risk asymmetry primarily at the **sender's authentication layer**; if the sender's credentials have been compromised (via SIM swap, phishing, or credential stuffing), every subsequent transaction they authorise appears operationally valid to the system.

A typical retail transaction carries the following data fields:

| Field | Business Meaning | Data Type |
|---|---|---|
| Originating Bank BIC | Sending institution identifier | Categorical |
| Beneficiary Bank BIC | Receiving institution identifier | Categorical |
| Sender Account Number | Masked account ID | String |
| Recipient Account Number | Masked account ID or phone alias | String |
| Transaction Amount (KES) | Monetary value | Continuous |
| Timestamp | Date, time, timezone | Temporal |
| Channel Code | USSD / Mobile / ATM / Web / Agent | Categorical |
| Transaction Reference | Unique system-assigned ID | String |
| Device IMEI (if mobile) | Hardware fingerprint | String |
| IP Address (if web/mobile) | Network location | Categorical |
| Session Token | Session identifier | String |
| Geolocation (if available) | Approximate physical location | Continuous pair |

### 3.2 Graph-Theoretic Representation

When the full corpus of PesaLink transactions is modelled mathematically, the payment network becomes a **directed, temporal, heterogeneous multi-relational graph**, formally defined as:

$$
\mathcal{G} = (\mathcal{V}, \mathcal{E}, \mathcal{R}, \mathbf{X}, \mathbf{Y}, T)
$$

where each component carries both a mathematical and a business interpretation:

#### 3.2.1 The Node Set $\mathcal{V}$: "Who is in the network?"

In graph theory, nodes (or vertices) are the **entities** of the domain. In PesaLink, the heterogeneous node set encompasses multiple entity types:

| Node Type | Graph Symbol | Business Identity | Example |
|---|---|---|---|
| **Account Node** | $v_a$ | A bank account held at any member institution | KCB Account 1234567890 |
| **Device Node** | $v_d$ | A registered smartphone (identified by IMEI) | Samsung Galaxy S24, IMEI 35XXXXXXX |
| **USSD Session Token** | $v_s$ | An ephemeral telco-assigned session identifier | Safaricom session #8821 |
| **ATM Terminal Node** | $v_t$ | A physical ATM or agent device | Equity Bank ATM, Westlands branch |
| **IP Subnet Node** | $v_n$ | A network block associated with a transaction | 197.248.x.x (Nairobi ISP) |
| **Geographic Cell Node** | $v_g$ | A telco base-station cell or GPS region | Kibera BTS #441 |
| **Merchant Node** | $v_m$ | A POS-registered business | Nakumatt Mega, Merchant ID 55221 |

This heterogeneity is the critical departure from traditional tabular fraud models. A single account node $v_a$ may be connected to multiple device nodes, indicating that the account is accessed from different physical handsets, a pattern strongly associated with **mule account syndicate** operations.

#### 3.2.2 The Edge Set $\mathcal{E}$: "What happened between entities?"

Edges encode **relationships** between nodes. In PesaLink, edges are directed (money flows from source to destination) and typed by the relation $r \in \mathcal{R}$:

| Edge Type (Relation $r$) | Graph Notation | Business Meaning |
|---|---|---|
| **Transfer edge** | $e_{ij}^{\text{transfer}}$ | Account $v_i$ sent funds to account $v_j$ |
| **Device-Account binding** | $e_{ad}^{\text{uses}}$ | Account $v_a$ was accessed using device $v_d$ |
| **Session-Account binding** | $e_{as}^{\text{initiates}}$ | Account $v_a$ initiated a USSD session $v_s$ |
| **Terminal-Account binding** | $e_{at}^{\text{withdraws}}$ | Account $v_a$ performed a cash withdrawal at terminal $v_t$ |
| **IP-Account binding** | $e_{an}^{\text{connects_from}}$ | Account $v_a$ was accessed from IP subnet $v_n$ |

Each transfer edge $e_{ij}$ carries an **edge feature vector**:

$$
\mathbf{e}_{ij} = [\text{amount}, \Delta t_{\text{since_last}}, \text{channel_code}, \text{sequence_rank}]
$$

where $\Delta t_{\text{since_last}}$ is the time elapsed since the previous transaction from the same source node, a critical temporal feature for detecting **velocity attacks** (automated high-frequency transfers designed to exhaust an account balance before fraud is detected).

#### 3.2.3 The Node Feature Matrix $\mathbf{X}$: "What do we know about each entity?"

Each node $v \in \mathcal{V}$ carries a feature vector $\mathbf{x}_v \in \mathbb{R}^d$ encoding its local attributes:

| Feature | Business Meaning | Fraud Signal |
|---|---|---|
| Account age (days) | How long the account has existed | New accounts disproportionately used in mule networks |
| Historical velocity score | Average transaction frequency over 30 days | Sudden spikes indicate compromise |
| KYC completeness flag | Whether all Know Your Customer documents are verified | Incomplete KYC linked to synthetic identity fraud |
| Average transaction amount | Baseline monetary behaviour | Structuring attacks stay below this threshold |
| Channel diversity index | Entropy of channels used | Fraudsters often favour one channel post-compromise |
| Dormancy flag | Account inactive for >90 days | Dormant accounts frequently recruited as mule recipients |
| Degree centrality | Number of direct transaction counterparties | High in-degree spikes indicate fund aggregation |

#### 3.2.4 Graph Structural Properties

Bank transaction graphs exhibit three well-documented structural properties that distinguish them from social networks and require specialised modelling approaches [6]:

**Sparsity**: If $N$ is the total count of PesaLink-enrolled accounts, the maximum possible directed edges equal $N \times (N-1)$. In practice, any single account interacts with a microscopic fraction of all other accounts; the adjacency matrix is **near-empty**. This sparsity demands storage in **Compressed Sparse Row (CSR)** or **Coordinate (COO)** format rather than dense matrix representation, and motivates graph-specific sampling strategies over full-graph convolution.

**Bipartite sub-structure**: A large fraction of transactions follow a **bipartite graph** topology. Accounts (one partition, $U$) transact with merchants or business entities (second partition, $M$), such that edges only cross between partitions: $e \in U \times M$. Pure person-to-person transfer subgraphs, by contrast, form a general directed graph with bidirectional possible edges within the single account partition.

**Scale-free degree distribution**: Like most real-world networks, the PesaLink transaction graph follows a power-law degree distribution. A small number of highly-connected "hub" accounts (typically high-volume merchants, payroll disbursement accounts, or utility providers) coexist with a vast majority of low-degree retail accounts. Fraud rings deliberately exploit this by inserting **mule accounts** at high in-degree positions to aggregate illicit inflows before dispersal.

**Temporal evolution**: Unlike static graphs, the PesaLink graph is a **dynamic temporal graph** $\mathcal{G}(T)$ where edge sets appear and disappear at timestamps $t \in T$. A fraud pattern invisible in a static snapshot may be clearly visible as a temporal subgraph; for example, ten inbound transfers arriving within 90 seconds followed by a single large outbound transfer constitute a temporal signature of a mule account being "loaded and emptied."

---

## 4. Example Transaction Cases and Graph Patterns

The following nine scenarios span the full breadth of PesaLink activity, from routine retail transfers to sophisticated cross-bank fraud rings. Each is described in both **business terms** (what is happening operationally) and **graph-theoretic language** (how the event is represented in the mathematical model $\mathcal{G}$). Legitimate cases are presented first to establish the baseline structural signatures against which anomalies are detected.

---

### Case 1 (Legitimate): Retail Peer-to-Peer Transfer: Rent Payment

**Business narrative**: Grace, a teacher at a public school in Nairobi, pays her monthly rent of KES 18,500 to her landlord James on the 1st of every month. Grace banks with KCB; James banks with Equity Bank. Grace opens her KCB mobile app, selects "Bank Transfer," selects Equity Bank as the destination, and enters James's account number. She confirms with her PIN. Behind the scenes, KCB's core banking system routes the instruction through the PesaLink switch, which validates the destination account with Equity Bank and confirms fund availability before committing the credit. The money arrives in James's account within 3 seconds. PesaLink itself is the shared network infrastructure; neither Grace nor James is directly aware of it. The two banks handle all routing and settlement via the switch in the background.

**Graph-theoretic description**:
- **Nodes involved**: $v_{\text{Grace}} \in \mathcal{V}_a$ (KCB account node), $v_{\text{James}} \in \mathcal{V}_a$ (Equity account node), $v_{d_G} \in \mathcal{V}_d$ (Grace's smartphone, IMEI stable).
- **Edges generated**: One directed transfer edge $e_{\text{Grace} \to \text{James}}^{\text{transfer}}$ with weight KES 18,500; one device-account binding edge $e_{\text{Grace},d_G}^{\text{uses}}$ (same device as prior 11 months).
- **Edge feature vector**: $\mathbf{e} = [18500,\ \Delta t \approx 30\ \text{days},\ \text{MOBILE},\ 1]$.
- **Node features**: Grace's account: age = 4.2 years, velocity score = low, channel diversity = 1 (mobile only), degree centrality = 12 known counterparties. James's account: high in-degree from multiple known tenants, a legitimate "hub" receiving monthly rent from several accounts.
- **Graph signature**: A periodic edge with stable inter-arrival time $\Delta t \approx 720\ \text{hours}$, fixed amount, consistent device, and a known bilateral link (shortest path $d(\text{Grace}, \text{James}) = 1$ in prior graph snapshots). This temporal regularity is a strong benign signal, the R-GCN assigns low anomaly score because the edge re-appears in an expected structural position within the graph.

---

### Case 2 (Legitimate): Enterprise Bulk Payroll Disbursement (B2P)

**Business narrative**: Twiga Foods, a Nairobi-based produce distribution company, processes payroll for 340 field agents across 11 banks on the 25th of each month. Their treasury team uploads a bulk payment file via the PesaLink Open Banking API, disbursing between KES 22,000 and KES 85,000 to each agent depending on their route and performance tier.

**Graph-theoretic description**:
- **Nodes involved**: $v_{\text{Twiga}} \in \mathcal{V}_a$ (corporate disbursement account, Standard Chartered Bank), plus 340 recipient account nodes $\{v_{r_i}\}_{i=1}^{340}$ across 11 institutions.
- **Edges generated**: 340 directed transfer edges $\{e_{\text{Twiga} \to r_i}^{\text{transfer}}\}$, all initiated within a 90-second batch window. One API session node $v_{s_{\text{API}}}$ binding edge.
- **Structural signature**: A **radial star subgraph** centred on $v_{\text{Twiga}}$ with 340 leaf nodes. All 340 recipient nodes have appeared as neighbours of $v_{\text{Twiga}}$ in at least 3 prior monthly snapshots. The out-degree burst ($\Delta \deg^{+} = 340$) is large but temporally predictable, a monthly spike at $t \approx 25\text{th}$ of every month.
- **Feature context**: Source account age = 7.1 years; average monthly out-flow = KES 14.2M (consistent with current batch); all recipients have non-zero degree in the prior graph (not dormant). The **temporal periodicity** of the burst, combined with the **familiar recipient set**, produces a benign structural embedding.
- **Model behaviour**: A GNN trained with Focal Loss will have encountered this pattern repeatedly in training. The node embeddings of all 340 recipients will carry high similarity scores to known legitimate payroll recipients. The anomaly probability output approaches zero.

---

### Case 3 (Legitimate): SACCO Loan Disbursement to Member Accounts

**Business narrative**: Stima SACCO disburses approved development loans to 28 member accounts across 6 different banks. Loan amounts range from KES 50,000 to KES 450,000. Disbursements are triggered after a credit committee approval logged internally, then executed via the SACCO's Internet Banking portal.

**Graph-theoretic description**:
- **Nodes**: $v_{\text{Stima}} \in \mathcal{V}_a$ (SACCO disbursement account, Cooperative Bank), $\{v_{m_i}\}_{i=1}^{28}$ (member accounts across KCB, Equity, NCBA, Family Bank, I\&M Bank, Absa).
- **Edges**: 28 directed transfer edges, batch-initiated via Internet Banking (channel code = WEB). One IP subnet node $v_{n}$ (Stima SACCO corporate network, static IP).
- **Key graph property**: This is a **bipartite-like subgraph** where one "institution node" (Stima SACCO) connects to multiple "individual account" nodes across institutions. The amounts are heterogeneous (varying loan sizes), unlike payroll which is often uniform, but all recipients have a prior history as SACCO members (i.e., prior edges linking them back to $v_{\text{Stima}}$ via repayment transactions in the reverse direction).
- **Reverse-edge evidence**: The graph contains prior edges $\{e_{m_i \to \text{Stima}}^{\text{repayment}}\}$ for most members, monthly loan repayment edges that establish a **bilateral relationship** in the graph. The GNN's message-passing reads both the forward (loan) and backward (repayment) edge history to confirm the relationship is legitimate.
- **Model behaviour**: The existence of historical reverse edges (loan repayments) provides strong contextual signal of a legitimate lender-borrower relationship. The R-GCN encodes this bidirectional structural pattern as a low-risk embedding.

---

### Case 4 (Legitimate): Recurring Utility Payment via USSD

**Business narrative**: Peter, a small-scale maize trader in Eldoret, pays his Kenya Power prepaid electricity token every 2–3 weeks using the Cooperative Bank USSD code \*667#. Each payment is between KES 500 and KES 1,200 depending on his business activity. He has been doing this for 3 years. Kenya Power's collection account is held at KCB.

**Graph-theoretic description**:
- **Nodes**: $v_{\text{Peter}} \in \mathcal{V}_a$ (Co-op Bank account), $v_{\text{KenyaPower}} \in \mathcal{V}_m$ (merchant/utility node at KCB), $v_{s_i} \in \mathcal{V}_s$ (ephemeral USSD session tokens, new per transaction).
- **Edges**: Recurring directed transfer edges $\{e_{\text{Peter} \to \text{KenyaPower}}^{\text{transfer}}\}$ at irregular but bounded intervals ($\Delta t \in [14, 21]$ days). New USSD session node and binding edge per transaction, but all originating from the same IMSI (Peter's SIM card).
- **Bipartite structure**: Peter's account participates in a **consumer-to-merchant bipartite subgraph** $U \times M$ where $U = \{\text{individual accounts}\}$ and $M = \{\text{utility/merchant accounts}\}$. Kenya Power's node $v_{\text{KenyaPower}}$ has extremely high in-degree, connected to tens of thousands of individual consumer accounts nationwide. This is a classic **hub node** in the scale-free graph.
- **Structural properties**: New USSD session tokens are ephemeral (short-lived nodes that appear and expire within the session). The model must learn that ephemeral $v_s$ nodes bound to a stable account $v_a$ with consistent IMSI are a benign structural pattern, the "diamond" subgraph $v_{\text{Peter}} - v_{s_i} - v_{\text{KenyaPower}}$ recurs predictably.
- **Model behaviour**: The high in-degree of $v_{\text{KenyaPower}}$ and the temporal regularity of Peter's edge re-appearance make this a near-zero-risk pattern in the GNN's embedding space.

---

### Case 5 (Legitimate): Cross-Bank Savings Transfer: Financial Planning

**Business narrative**: Dr. Amina, a consultant in Nairobi, maintains a current account at Absa Bank for daily operations and a high-yield savings account at NCBA for long-term savings. On the last working day of each month, she transfers KES 75,000 from her Absa account to her NCBA savings account via her Absa mobile app. She has maintained this pattern for 22 months.

**Graph-theoretic description**:
- **Nodes**: $v_{\text{Absa}} \in \mathcal{V}_a$ (Dr. Amina's Absa account), $v_{\text{NCBA}} \in \mathcal{V}_a$ (Dr. Amina's NCBA savings account, a distinct node despite the same beneficial owner), $v_{d_A}$ (Dr. Amina's iPhone, stable IMEI).
- **Edges**: Monthly directed transfer edge $e_{\text{Absa} \to \text{NCBA}}^{\text{transfer}}$ with highly consistent amount (KES 75,000 ± 0) and inter-arrival time ($\Delta t \approx 30$ days ± 2 days for working-day adjustments).
- **Key observation, same-owner multi-account pattern**: The graph does not inherently know that both account nodes belong to the same individual. What it observes is a stable bilateral link with a single, predictable periodic edge in one direction and no reverse-flow (Dr. Amina does not withdraw from her savings account frequently). This **asymmetric, periodic single-direction edge** is a distinctive structural signature of savings-transfer behaviour versus fraud (which typically shows rapid round-trips).
- **Feature context**: Both account nodes have age \gt 2 years; the link $e_{\text{Absa} \to \text{NCBA}}$ has appeared 22 times; the device node $v_{d_A}$ is consistent; IP subnet is consistent (Amina's home WiFi block). Degree centrality of $v_{\text{NCBA}}$ = 1 (Dr. Amina's Absa account is its only counterparty), a structural isolation pattern not associated with mule accounts (which receive from many diverse sources).
- **Model behaviour**: The model's 2-hop subgraph around $v_{\text{NCBA}}$ contains only one transfer-edge neighbour and no unusual device or IP bindings, producing a tightly consistent, low-entropy embedding that scores near-zero for anomaly.

---

### Case 6 (Legitimate): NGO Beneficiary Cash Transfer Programme

**Business narrative**: A humanitarian organisation managing a drought-response cash transfer programme in Turkana County disburses KES 5,000 monthly to 2,400 registered beneficiaries. Beneficiaries hold accounts at microfinance institutions (MFIs) and SACCOs connected to PesaLink. Disbursements are executed via the NGO's treasury account at I&M Bank through the Bulk Payments API.

**Graph-theoretic description**:
- **Nodes**: $v_{\text{NGO}} \in \mathcal{V}_a$ (I&M Bank treasury account), $\{v_{b_i}\}_{i=1}^{2400}$ (beneficiary accounts across multiple MFIs and SACCOs). All 2,400 recipient nodes were pre-registered in a KYC programme, their node features include a KYC completeness flag = 1 and a "programme beneficiary" metadata tag.
- **Edges**: 2,400 directed transfer edges, all amount = KES 5,000 (perfectly uniform), executed within a single batch window of approximately 8 minutes, channel = bulk API.
- **Structural signature**: This generates the **largest single-event star subgraph** in the PesaLink graph, a degree burst of 2,400 from a single source. The uniform amount and tight temporal batch distinguish it from a fraud "fan-out" pattern.
- **Critical distinction from fraud**: A smurfing fan-out (Case 8 below) typically exhibits heterogeneous amounts (varying to stay below thresholds), a sequential rather than simultaneous timing pattern, and recipient nodes with zero prior transaction history. This NGO pattern has uniform amounts, simultaneous batch execution, and recipients with prior registration metadata.
- **Model behaviour**: The combination of uniform amounts, pre-registered recipient embeddings, and historical repetition (monthly recurrence) produces a benign "programme disbursement" cluster in the GNN's latent embedding space, well-separated from fraud ring embeddings.

---

### Case 7 (Fraud): SIM Swap: Identity Hijacking with Device Substitution

**Business narrative**: A fraudster bribes a Safaricom agent outlet in Mombasa to execute an illegal SIM replacement for Bernard's registered mobile number. With the new SIM, they receive Bernard's OTP, log into his Family Bank mobile app from a different handset, and within 6 minutes execute three transfers: KES 390,000 to a KCB account, KES 380,000 to an Equity account, and KES 199,000 to an NCBA account. All three recipient accounts were opened within the past 14 days and have no prior transaction history.

**Graph-theoretic description**:
- **Device substitution event** (temporal graph change at $t_0$): The edge $e_{\text{Bernard},d_{\text{old}}}^{\text{uses}}$ (Bernard's Samsung Galaxy, stable IMEI for 2.7 years) becomes inactive. Simultaneously, a new device node $v_{d_{\text{new}}}$ (unseen IMEI) appears and binds to $v_{\text{Bernard}}$ via edge $e_{\text{Bernard},d_{\text{new}}}^{\text{uses}}$.
- **Transfer fan-out** (at $t_0 + 90\text{s}$, $t_0 + 210\text{s}$, $t_0 + 370\text{s}$): Three directed transfer edges $e_{\text{Bernard} \to r_1}^{\text{transfer}}$, $e_{\text{Bernard} \to r_2}^{\text{transfer}}$, $e_{\text{Bernard} \to r_3}^{\text{transfer}}$ where:
  - $r_1, r_2, r_3$ have degree centrality = 0 in prior graph snapshots (dormant).
  - The $k$-hop shortest path $d(\text{Bernard}, r_i)$ = ∞ in all prior snapshots.
  - Amounts: KES 390K, 380K, 199K, deliberately structured below the KES 500K enhanced scrutiny threshold.
- **Velocity signature**: $\Delta t_{\text{since_last}}$ for Bernard's account = 18 days (he transacts infrequently). The inter-transaction time within this fraud burst = 90s and 160s, an extreme temporal compression against the historical baseline.
- **2-hop neighbourhood analysis**: At $t_0 + 370\text{s}$, the 2-hop neighbourhood of $v_{\text{Bernard}}$ suddenly includes three new dormant leaf nodes connected via high-amount edges, structurally indistinguishable from a mule network aggregation target. The R-GCN flags this as a high-anomaly embedding.
- **What a single-bank rule misses**: Each individual transfer (KES 390K, 380K, 199K) is below the automated blocking threshold. The rule system sees three "valid" transactions. The GNN sees a new device binding at $t_0$, a velocity explosion from $\Delta t = 18\ \text{days}$ to $\Delta t = 90\ \text{seconds}$, and three structural links to zero-history nodes, collectively a known fraud motif.

---

### Case 8 (Fraud): Smurfing and Multi-Layer Structuring (Cross-Bank)

**Business narrative**: A narcotics trafficking syndicate receives KES 4.8 million in illicit cash from street-level collectors, deposited in mixed-amount agent banking transactions into a "front" account (Account A) at Absa Bank. To launder the funds through the banking system, the syndicate has recruited 18 "mules", individuals who opened accounts across 7 different banks using genuine but misappropriated National ID documents. The funds are split and moved in a two-hop laundering pattern.

**Hop 1: Distribution** (within 25 minutes of Account A receiving full balance):
Account A sends 18 transfers ranging from KES 200,000 to KES 249,000 to accounts $\{m_i\}_{i=1}^{18}$ across 7 banks, all deliberately below KES 250,000 (the threshold for enhanced transaction monitoring at several institutions).

**Hop 2: Convergence** (within 40 minutes of Hop 1):
Each mule account $m_i$ sends a single transfer to one of two "collection" accounts ($C_1$ at KCB, $C_2$ at Equity Bank), aggregating the dispersed funds for final withdrawal.

**Graph-theoretic description**:
- **Hop 1 subgraph**: Directed fan-out edges $\{e_{A \to m_i}^{\text{transfer}}\}_{i=1}^{18}$. Source $v_A$ acquires an out-degree burst of 18. All $m_i$ nodes: dormancy flag = 1, KYC completeness = partial (misappropriated IDs), prior degree = 0.
- **Hop 2 subgraph**: 18 directed edges $\{e_{m_i \to C_j}^{\text{transfer}}\}$ converging on $v_{C_1}$ and $v_{C_2}$. This creates a **convergent directed path subgraph**, a butterfly or hourglass topology where 18 nodes funnel into 2.
- **Complete 2-hop topology**: $v_A \to \{v_{m_i}\} \to \{v_{C_1}, v_{C_2}\}$, a **directed bipartite relay graph** with depth 2. The full topology is only visible by combining the local subgraph of 7 different banks. No single institution observes more than 2–3 $m_i$ nodes connecting to a local $C_j$.
- **Critical federated learning implication**: This is the canonical use case for cross-bank graph visibility. Each individual bank's local model sees a small, locally innocuous fan-out or fan-in. Only the federated model, aggregating structural knowledge from all 7 banks, reconstructs the complete butterfly topology and identifies the coordinated relay pattern.
- **Temporal signature**: The entire Hop 1 → Hop 2 sequence completes in under 65 minutes. In temporal graph analysis, this creates a directed **causal temporal path** $v_A \xrightarrow{t_0} v_{m_i} \xrightarrow{t_0 + \delta} v_{C_j}$ where $\delta \lt 40\ \text{minutes}$, a rapid relay that distinguishes laundering from legitimate multi-step transfers (which typically have inter-hop intervals of days to weeks).

---

### Case 9 (Ambiguous / False-Positive Risk): Emergency Family Transfer

**Business narrative**: Collins, a Nairobi resident, receives an urgent call from his mother in Kisumu that a family member has been hospitalised. He immediately transfers KES 150,000 from his Equity Bank account to his brother Daniel's KCB account (a number Collins has not transacted with before on PesaLink) and an additional KES 80,000 to a family friend's Cooperative Bank account for hospital supplies. Both transfers happen within 8 minutes of each other, from Collins's usual device.

**Graph-theoretic description**:
- **New counterparty edges**: $e_{\text{Collins} \to \text{Daniel}}^{\text{transfer}}$ and $e_{\text{Collins} \to \text{FriendAcct}}^{\text{transfer}}$, both recipient nodes have shortest-path distance $d \gt 1$ from $v_{\text{Collins}}$ in prior history (never directly transacted before, though Collins may share 2-hop neighbours with Daniel via other family transactions).
- **Device consistency** (key distinction from fraud): The device node $v_{d_C}$ (Collins's regular Samsung, IMEI stable for 3 years) is unchanged. There is no device substitution event. The same IP subnet and geolocation cell are observed.
- **Velocity anomaly**: $\Delta t_{\text{since_last}} = 4\ \text{days}$ → two transfers within 8 minutes, a compression from Collins's normal behaviour. The velocity score spikes.
- **Amount anomaly**: KES 150,000 and KES 80,000 are significantly above Collins's 30-day average transaction size of KES 22,000.
- **Why this could score highly for fraud**: New counterparties + velocity compression + above-average amounts. A rule-based system would almost certainly block or flag these transfers.
- **Why the GNN should not classify it as fraud**: The 2-hop subgraph reveals that Daniel's account is connected (via prior transactions) to other nodes in Collins's neighbourhood, family members who share a common transaction history. The device node $v_{d_C}$ has zero anomaly (no substitution). The amounts, while large, are within the account's historical maximum (Collins made a KES 200,000 property deposit 14 months ago). The GNN's attention mechanism, conditioned on the full neighbourhood embedding, suppresses the false positive by weighing the device stability and 2-hop relationship against the velocity signal.
- **Modelling lesson**: This case illustrates why graph-based models produce fewer false positives than rule-based systems. The structural context, stable device, familiar neighbourhood connectivity at 2 hops, provides exculpatory signal that a transaction-level tabular model cannot access.

---

## 5. The Fraud Landscape: CBK Statistics and Regulatory Response

### 5.1 CBK Quantitative Fraud Metrics

According to the **Central Bank of Kenya Bank Supervision Annual Report 2024** and the associated **Financial Sector Stability Report**, the following escalation in fraud was recorded across the banking sector [2]:

| Metric | 2023 | 2024 | Year-on-Year Change |
|---|---|---|---|
| Total Reported Fraud Cases | 153 | 353 | **+130.72%** |
| Total Financial Exposure Value | KES 680.9 million | KES 1.90 billion | **+179.04%** |
| Net Financial Defalcation Losses | KES 412.0 million | KES 1.50 billion | **+264.08%** |
| Card-Specific Fraud Losses | KES 15.50 million | KES 263.29 million | **+1,598.65%** |
| Identity Theft / Impersonation Losses | KES 33.16 million | KES 199.00 million | **+500.12%** |
| Mobile Banking Vector Incidents | Baseline | 146 cases | N/A |
| Online/Web Banking Vector Incidents | Baseline | 106 cases | N/A |

The card-specific fraud loss trajectory (+1,598.65% in one year) is particularly alarming and reflects the systematic exploitation of the contactless payment expansion and the growth of e-commerce payment channels linked to bank accounts via PesaLink's open API infrastructure.

### 5.2 Prevalent Fraud Modalities

**SIM Swap and Identity Hijacking**: Fraudsters execute illegal SIM card replacements via telecommunications retailer networks, intercepting OTPs to authenticate digital banking sessions. Once in control of the victim's registered mobile number, they perform credential resets and initiate high-velocity outbound transfers. In graph terms, this produces a **sudden device node substitution**, the account node $v_a$ disconnects from its historical device node $v_{d_{\text{old}}}$ and immediately binds to a new, previously-unseen device node $v_{d_{\text{new}}}$.

**Social Engineering and Vishing**: Attackers impersonate regulatory officials, law enforcement, or bank staff to manipulate victims into self-authorising transfers. Unlike technical exploits, this attack vector produces no anomalous device binding, the legitimate device and credentials are used. However, it produces **recipient anomalies**: the transfer targets accounts with zero historical connection to the sender ($k$-hop shortest path = ∞ in prior transaction history).

**Smurfing and Layering Networks (Structuring)**: Criminal syndicates move funds across multiple destination accounts in small increments, deliberately staying below AML monitoring thresholds (typically KES 500,000 for enhanced scrutiny). In graph terms, this creates a **fan-out subgraph** from a single source to multiple dormant recipient nodes, a pattern that becomes clearly anomalous when the temporal density (multiple edges within seconds from one node) is modelled alongside the structural topology.

**Mule Account Syndicates**: Networks of dormant or fraudulently-opened accounts receive illicit funds which are then rapidly withdrawn via agents or ATMs. Mule nodes exhibit a distinctive **in-degree spike followed by rapid out-degree** in temporal graph analysis: high in-flow velocity, followed by a single terminal withdrawal edge at an agent node $v_t$.

**Insider Collaboration**: Bank personnel abuse administrative privileges to bypass transaction limits or AML alerts. Forensic investigations suggest insider complicity in a notable proportion of high-value digital banking breaches [2]. In graph terms, insider attacks may be invisible to transaction-level models but detectable at the **administrative action graph** level, a secondary graph tracking system actions correlated with subsequent unusual transactions.

### 5.3 Judicial Pressure: The DTB-Safaricom Landmark Ruling

On 13 July 2026, the **High Court of Kenya at Machakos** upheld a ruling that fundamentally reframes institutional liability for digital fraud [3]. In a case involving coordinated SIM swap fraud resulting in the theft of KES 4.42 million, the court rejected the standard automated defence that "the transaction used the correct PIN and occurred within system limits." The court ruled that:

1. The **velocity and anomalous pattern** of transactions, multiple high-value transfers within minutes, constituted a detectable fraud signature that the institutions' systems should have flagged.
2. Financial institutions are legally obligated to deploy **predictive, behaviour-aware fraud detection**, not merely reactive rule-based blocking.
3. Liability was apportioned: Safaricom (as the compromised telecommunications provider) bears **60%** and Diamond Trust Bank (as the financial institution) bears **40%** of the total loss.

This ruling creates a binding legal precedent that transforms graph-based anomaly detection from a technical enhancement into a **regulatory compliance imperative**.

---

## 6. Data Privacy Constraints: The ODPC Framework

### 6.1 The Kenya Data Protection Act 2019

The **Data Protection Act, No. 24 of 2019 (DPA)** establishes the legal framework governing the processing of personal data in Kenya, enforced by the **Office of the Data Protection Commissioner (ODPC)** [7]. Financial institutions operating on the PesaLink network are classified simultaneously as **data controllers** (determining the purpose and means of processing) and **data processors** (actually performing the processing), subjecting them to the full scope of statutory obligations.

Key provisions with direct architectural implications for fraud detection systems:

**Section 25, Principles of Data Protection (Data Minimisation)**:  
Personal data must be "adequate, relevant, and limited to what is necessary in relation to the purposes for which they are processed." A centralised fraud model that requires banks to pool raw customer transaction records, account histories, and personal identifiers into a shared repository violates this principle when technically less invasive alternatives, specifically Federated Learning, are available.

**Section 41, Data Protection by Design and by Default**:  
Data controllers must implement technical and organisational measures that "integrate data protection into processing activities." Centralised data pooling creates a single point of failure, a "honey pot" that, if breached, exposes the transaction history of the entire network. Privacy by Design mandates that the architecture intrinsically prevents unnecessary data aggregation from the outset.

**Section 48, Cross-Border Data Transfers**:  
Personal data may only be transferred outside Kenyan jurisdiction with explicit authorisation from the ODPC or where the recipient country provides adequate protection. This provision effectively prohibits the use of standard global public cloud architectures (AWS US-East, GCP us-central1, etc.) for centralised model aggregation, requiring either in-country data residency or on-premises TEE-based solutions.

**Penalties**: Non-compliance carries fines of up to **KES 5 million or 1% of annual gross turnover**, whichever is higher, a material operational risk for any major commercial bank.

### 6.2 CBK Prudential Guidelines on Risk Management

The CBK's **Prudential Guidelines on Risk Management and Corporate Governance** impose direct operational requirements on the fraud detection function [4]:

- **Section 3.3 (Operational Risk Management)**: Banks must maintain robust internal control structures capable of mitigating technological threats. The CBK explicitly treats cyber fraud as a material threat to capital adequacy, fraud losses must be provisioned as operational risk exposure and reported in capital adequacy calculations.

- **FATF Grey-Listing Response**: Kenya's placement on the **Financial Action Task Force (FATF) grey list** (reflecting systemic Anti-Money Laundering and Counter-Financing of Terrorism deficiencies) compelled the CBK to require that all supervised institutions implement **automated, real-time transaction monitoring** with demonstrable anomaly detection capabilities. Manual post-hoc review is no longer acceptable as a primary control.

### 6.3 The Regulatory Paradox

The intersection of these two regulatory frameworks creates a structural paradox that defines the core problem this research addresses:

- The **CBK** demands real-time, automated, cross-bank-aware fraud detection to combat multi-institution fraud rings.
- The **ODPC/DPA** prohibits the centralised pooling of raw transaction data needed to build such cross-bank models.

Conventional fraud detection architectures require one of these mandates to be violated. A centralised model aggregating data from all PesaLink member banks achieves the CBK's detection requirement but breaches the DPA's data minimisation and privacy-by-design obligations. Conversely, fully siloed bank-level models satisfy the DPA but create the structural blind spots that allow cross-bank fraud rings, like the smurfing networks described above, to operate undetected.

---

## 7. The Case for Graph-Based Federated Learning

### 7.1 Why Tabular Models Fail

Traditional fraud detection, rule-based systems and tabular machine learning models (logistic regression, gradient-boosted trees, standard neural networks), evaluate each transaction as an isolated event, producing a fraud probability score based solely on the transaction's own feature vector. This approach is blind to **relational anomalies**: it cannot detect a mule account that receives 30 individually-normal transfers in 60 seconds, nor can it identify a sender whose device has just changed for the first time in three years.

Furthermore, PesaLink transactions exhibit **extreme class imbalance**: fraudulent transactions account for less than 0.1% of total network volume. However, as evidenced by the CBK statistics documenting over KES 1.50 billion in net losses, although these fraudulent cases are incredibly infrequent, their individual financial severity and cumulative economic damage are massive. Standard cross-entropy loss functions catastrophically fail in this regime, as a model achieves artificially high accuracy (>99.9%) by classifying every transaction as legitimate, a strategy that generates no fraud detections at all. **Focal Loss** [8], which dynamically suppresses the gradient contribution of easy-to-classify legitimate transactions and forces learning on ambiguous fraud signals, is the required objective function.

### 7.2 Why Graph Neural Networks Are the Right Model Class

Graph Neural Networks (GNNs) directly address the relational blindness of tabular models. By operating on the transaction graph $\mathcal{G}$, a GNN propagates information between structurally connected nodes through **message passing**, each node aggregates representations from its neighbourhood:

$$
h_v^{(k)} = \sigma\left(W^{(k)} \cdot \text{CONCAT}\left(h_v^{(k-1)},\ \text{AGGREGATE}\left(\{h_u^{(k-1)} : u \in \mathcal{N}(v)\}\right)\right)\right)
$$

where $\mathcal{N}(v)$ is the set of neighbouring nodes, $W^{(k)}$ is the learnable weight matrix at layer $k$, and $\sigma$ is a non-linear activation. After $K$ layers, each node's representation $h_v^{(K)}$ encodes structural information from its $K$-hop neighbourhood, enabling the model to detect the multi-hop mule ring topology described in Case 3 above.

For the heterogeneous multi-relational PesaLink graph (multiple node types, multiple edge types), **Relational Graph Convolutional Networks (R-GCNs)** [9] extend this formulation:

$$
h_i^{(l+1)} = \sigma\left(W_0^{(l)} h_i^{(l)} + \sum_{r \in \mathcal{R}} \sum_{j \in \mathcal{N}_i^r} \frac{1}{c_{i,r}} W_r^{(l)} h_j^{(l)}\right)
$$

where $W_r^{(l)}$ is a relation-specific weight matrix capturing the semantics of each edge type (transfer, device binding, IP connection, etc.). This allows the model to learn that a device-binding edge conveys different fraud information from a transfer edge.

**Graph Attention Networks (GAT)** [10] further refine this by replacing the uniform aggregation with a learnable attention mechanism, each neighbour's contribution is weighted by an attention coefficient $\alpha_{ij}$ computed from the feature similarity between nodes $i$ and $j$. In a PesaLink fraud context, this means the model learns to pay more attention to high-frequency counterparties and less to isolated, one-off transfers, without requiring manual feature engineering.

For scalability to the full PesaLink network, which handles over 500,000 transactions per month [5] with a graph size growing continuously, **ClusterGCN** [11] partitions the global transaction graph using the METIS algorithm into dense local sub-communities:

$$
V = [V_1, V_2, \dots, V_c]
$$

Training proceeds on individual cluster batches, with intra-cluster edges preserved and inter-cluster edges dynamically restored through stochastic multi-cluster batching. This eliminates the "neighbourhood explosion" problem (where computing a single node's embedding requires fetching exponentially growing numbers of neighbours) and enables training on graphs with millions of nodes on bounded memory hardware.

### 7.3 Why Federated Learning is the Required Training Paradigm

Standard GNN training requires access to the full graph $\mathcal{G}$, which, in the cross-bank PesaLink context, means aggregating transaction data from all 80+ member institutions into a centralised repository. This is precisely what the DPA prohibits.

Federated Learning (FL) resolves this by inverting the training paradigm: data stays within each bank's secure infrastructure, and only encrypted model parameter updates (gradients or weight deltas) traverse the network to a central IPSL aggregation engine. The formal objective of the Federated Averaging (FedAvg) algorithm [12] is to minimise the global loss function:

$$
\min_w \sum_{k=1}^{K} \frac{n_k}{N} L_k(w)
$$

where $K$ is the number of participating banks, $n_k$ is the number of transaction nodes held by bank $k$, $N = \sum n_k$ is the total node count, and $L_k(w)$ is bank $k$'s local fraud detection loss. Each bank trains on its local graph $\mathcal{G}_k$, uploads its updated local weights $w_{t+1}^k$, and the coordinator computes:

$$
w_{t+1} = \sum_{k=1}^{K} \frac{n_k}{N} w_{t+1}^k
$$

No raw transaction record, account identifier, or customer profile leaves any bank's perimeter.

### 7.4 Graph Shapley Values and Monte Carlo Approximation for Contribution Tracking

Federated Learning introduces a structural free-rider problem: a malicious or low-quality participant bank can submit poisoned, noisy, or stale gradient updates that degrade the global model's accuracy, yet continue to benefit from the improved global weights without contributing meaningfully. To enforce fair contribution accounting and provide an automated data-poisoning defence, the platform computes **Graph Shapley Values** for each participant's gradient update at the end of every training round.

#### 7.4.1 The Shapley Value as a Fair Attribution Measure

Shapley values originate in cooperative game theory [22]. Given a set of $K$ participating banks and a model performance function $v: 2^{[K]} \to \mathbb{R}$ (e.g., global AUC-ROC on a held-out validation set), the Shapley value $\phi_k$ of bank $k$ is defined as its average **marginal contribution** across all possible orderings in which banks could have joined the federation:

$$
\phi_k = \sum_{S \subseteq [K] \setminus \{k\}} \frac{|S|! \, (K - |S| - 1)!}{K!} \Big[ v(S \cup \{k\}) - v(S) \Big]
$$

where $S$ ranges over all subsets of banks excluding $k$, and $v(S \cup \{k\}) - v(S)$ is the marginal improvement in global model performance attributable to including bank $k$'s gradient update in coalition $S$. In the PesaLink context, the characteristic function $v$ is evaluated on a privacy-safe global validation partition: the coordinator's TEE enclave reconstructs the global model for each coalition and measures AUC-ROC on a synthetic or public benchmark transaction set, without requiring any raw bank data.

This formulation guarantees three properties that make it suitable for a multi-bank regulatory context: (1) **Efficiency**: the sum of all Shapley values equals the total model improvement; (2) **Symmetry**: two banks whose gradient updates produce identical marginal improvements receive identical scores; and (3) **Null player**: a bank whose updates contribute nothing receives $\phi_k = 0$. Ghorbani and Zou (ICML 2019) [23] demonstrated that this metric is the unique fair data valuation measure satisfying all three axioms simultaneously, and showed that Monte Carlo estimation of Shapley values is both statistically consistent and computationally tractable for neural network models.

#### 7.4.2 The Computational Challenge: NP-Hardness and Graph Complexity

Exact computation of Shapley values requires evaluating $v(S)$ for every subset $S \subseteq [K]$: that is $2^K$ evaluations. For $K = 80$ member banks (the current PesaLink network size), this is $2^{80} \approx 10^{24}$ coalition evaluations, computationally infeasible. The challenge is further compounded by the **graph-structured nature** of the updates: each bank's local gradient is produced by a GNN operating on a local transaction subgraph $\mathcal{G}_k$, and the marginal value of that update depends on the structural properties of $\mathcal{G}_k$ (its density, its degree distribution, the number of fraud-positive nodes it contains) relative to all other banks' subgraphs. Two banks whose local graphs share many structural properties contribute less marginal information when added together than two banks whose subgraphs are topologically complementary.

#### 7.4.3 Monte Carlo Approximation with Guided Truncation (GTG-Shapley)

The platform implements the **GTG-Shapley (Guided Truncation Gradient Shapley)** algorithm [24], an approximation method that reconstructs partial federated models from gradient updates rather than retraining from scratch for each coalition, and applies guided Monte Carlo sampling with within-round and between-round truncation to dramatically reduce the number of evaluations required.

The GTG-Shapley procedure at each aggregation round $t$ proceeds as follows:

1. **Gradient-Based Model Reconstruction**: Instead of running a full FedAvg training loop for each coalition $S$, the coordinator's Intel SGX enclave reconstructs an approximate coalition model $\hat{w}_S$ by applying the weighted sum of gradient updates from the banks in $S$ to the prior round's global model:
$$
\hat{w}_S = w_t + \sum_{k \in S} \Delta w_k
$$
This avoids $O(2^K)$ full training runs, reducing reconstruction cost to $O(K)$ gradient additions per sample.

2. **Guided Monte Carlo Sampling**: Instead of sampling coalitions $S$ uniformly at random, the algorithm guides sampling towards coalitions that are most likely to produce high-variance Shapley estimates, i.e., coalitions where the marginal contribution of the next sampled bank is expected to be large. This is implemented via an Upper Confidence Bound (UCB) selection policy, borrowed directly from the **UCT (Upper Confidence Bound applied to Trees)** variant of Monte Carlo Tree Search [22], which balances the exploration of undersampled coalitions with the exploitation of coalitions known to produce informative marginal values:
$$
\text{UCB}(k, S) = \hat{\phi}_k^{(S)} + c \sqrt{\frac{\ln N_S}{n_{k,S}}}
$$
where $\hat{\phi}_k^{(S)}$ is the current estimated Shapley contribution of bank $k$ given coalition $S$, $N_S$ is the total number of samples drawn so far, $n_{k,S}$ is the number of times bank $k$ has been sampled in coalition $S$, and $c$ is an exploration coefficient. This UCT selection policy is precisely Monte Carlo Tree Search applied to the Shapley coalition space, the "tree" being the lattice of subsets $2^{[K]}$, with each level representing a coalition of a given size.

3. **Within-Round Truncation**: If the reconstructed model $\hat{w}_S$ produces performance $v(\hat{w}_S)$ that falls below a truncation threshold $\tau$ (e.g., AUC-ROC \lt 0.50), the coalition is truncated and assigned $\phi_k = 0$ for the marginal bank, without further evaluation. This eliminates computationally expensive evaluations of low-quality or malicious coalitions.

4. **Between-Round Warm-Starting**: Shapley estimates from round $t-1$ initialise the UCT tree for round $t$, so that estimation converges faster as the federation matures and gradient update quality stabilises.

Empirical results from the GTG-Shapley paper [24] demonstrate that this procedure approximates true Shapley values to within 5% relative error using as few as $O(K \log K)$ model evaluations, reducing the computational cost from $O(2^{80})$ to approximately $O(80 \times 7) \approx 560$ evaluations per round in the PesaLink scenario.

#### 7.4.4 Reputation Scoring and Data-Poisoning Defence

The Shapley value $\phi_k^{(t)}$ computed at each round $t$ feeds into an **exponential moving average reputation score** for bank $k$:

$$
\rho_k^{(t)} = \lambda \cdot \rho_k^{(t-1)} + (1 - \lambda) \cdot \phi_k^{(t)}
$$

where $\lambda \in (0, 1)$ is a decay parameter that weights recent contributions more heavily. Banks with $\rho_k$ consistently near zero or negative (i.e., whose updates degrade global performance) are flagged for review; banks whose scores fall below a hard threshold $\rho_{\min}$ are automatically de-authorised from the next aggregation round by the coordinator's Hyperledger Besu smart contract, which emits a `ParticipantSlashed` event and decrements that bank's reputation stake. This mechanism implements a **Sybil-resistant, incentive-compatible federated network**, providing formal game-theoretic guarantees that rational participant banks maximise their own Shapley score (and thus their reputation) by submitting honest, high-quality gradient updates derived from genuine transaction data.

### 7.5 Privacy Amplification: Differential Privacy, SMPC, and TEEs

Federated Learning alone does not constitute sufficient privacy protection, gradient updates can be reverse-engineered using **Deep Leakage from Gradients (DLG)** attacks to reconstruct the underlying training data [13]. The complete privacy stack requires:

- **Differential Privacy (DP-SGD)** [14]: Gaussian noise calibrated to the (ε, δ)-privacy budget is injected into local gradient updates before transmission, mathematically preventing membership inference attacks. The noise is calibrated using gradient clipping to sensitivity bound $C$ and noise multiplier $\sigma$:
$$
\tilde{g} = \frac{1}{B}\left(\sum_{i \in B} \frac{g_i(x_i)}{\max(1, \|g_i(x_i)\|_2 / C)} + \mathcal{N}(0, \sigma^2 C^2 \mathbf{I})\right)
$$

- **Secure Multi-Party Computation (SMPC)** [15]: Local weight updates are decomposed into cryptographic secret shares before transmission. No individual aggregation node can reconstruct any bank's gradient contribution in isolation, providing information-theoretic security against a compromised aggregation hub.

- **Optional Extension: zk-SNARKs for Verifiable Gradient Integrity in Decentralised (P2P) Deployments** [21]: In scenarios where participating banks refuse to grant IPSL or the KBA any coordination role, the central aggregator can be eliminated entirely in favour of a Peer-to-Peer (P2P) Gossip-based or ring-topology Federated Learning protocol. In this configuration, there is no trusted third party to verify that gradient updates submitted by each bank are honest and correctly clipped. zk-SNARKs (Zero-Knowledge Succinct Non-Interactive Arguments of Knowledge) address this verification gap. Before a bank transmits its SMPC gradient shares to its ring peers, its Intel TDX enclave generates a zk-SNARK proof demonstrating, in zero-knowledge, that: (1) the secret shares reconstruct to a gradient vector bounded within the DP-SGD clipping norm $C$; (2) the gradient was computed using the authorised GNN architecture; and (3) no individual training record was used more than once (replay protection). The receiving peer verifies the proof via a permissioned Hyperledger Besu smart contract using the `ecPairing` BN-254 precompile, requiring under 5 milliseconds. **This entire zk-SNARK generation and on-chain verification pipeline runs asynchronously, off the critical transaction settlement path.** No proof is generated or verified inline during PesaLink's sub-50 ms clearing window. Instead, batches of cleared transactions are grouped hourly, and a single aggregated batch proof is submitted to the ledger for regulatory auditability under CBK and ODPC guidelines. The ISO 20022 clearing payload carries no cryptographic proof data, preserving microsecond-level transport execution.

- **Trusted Execution Environments: Intel TDX at the Bank Node** [16]: At each participating bank, the local GNN training process, DP-SGD clipping, and SMPC share generation run inside an **Intel Trust Domain Extensions (TDX)** enclave. TDX operates at the Virtual Machine level, creating a hardware-isolated Trust Domain whose memory pages are encrypted by the CPU using AES-XTS before they are written to DRAM. Unlike legacy SGX, which confines individual application threads to small Processor Reserved Memory regions (typically 128 MB or 256 MB), TDX can allocate gigabytes of encrypted RAM, accommodating the full PyTorch Geometric graph data structures and GraphBLAS-based SpMM kernels without page-swapping overhead. This is critical because Kenya's participating banks range from Tier-1 institutions handling tens of millions of monthly transactions to SACCOs and MFIs with more modest but equally heterogeneous graph workloads. TDX's VM-level boundary also requires zero code refactoring, enabling rapid deployment across the diverse core banking platforms (T24, Flexcube, Finacle) in use across the IPSL member network. The central IPSL aggregation coordinator uses **Intel SGX** for fine-grained key management and zk-SNARK proof verification, where its tighter process-level trust boundary is the appropriate control. Together, this hybrid TDX (bank nodes) and SGX (coordinator) deployment satisfies the ODPC's Privacy by Design mandate under Section 41 of the DPA 2019, ensuring that raw gradients never exist in unencrypted form outside a hardware trust boundary.

---

## 8. Formal Problem Statement

The research problem addressed by this work can be stated as follows:

> **Given** a dynamic, temporal, heterogeneous multi-relational transaction graph $\mathcal{G}(T) = (\mathcal{V}, \mathcal{E}, \mathcal{R}, \mathbf{X}, \mathbf{Y}, T)$ distributed across $K$ member institutions (stratified into Tier-1, Tier-2, and Tier-3 entities) of the PesaLink/IPSL network, where:
> - each institution $k$ holds a local subgraph $\mathcal{G}_k$ that is **not permitted to be shared** under Kenya DPA 2019 Sections 25 and 41,
> - the global fraud detection objective requires **cross-institution subgraph visibility** to detect multi-hop fraud ring topologies (smurfing, mule networks, velocity attacks) that span institutional boundaries,
> - inference must complete in **under 50 milliseconds** to satisfy real-time settlement constraints. Because light travels at ~200km/ms in fiber, this mathematically restricts deployment away from standard public clouds (e.g., South Africa or Europe) and mandates **Bare-Metal Colocation** within the Nairobi Ring (Tier III/IV data centers like iXAfrica or ADC NBO1) connected via the Kenya Internet Exchange Point (KIXP),
> - and model parameters must provide **mathematically provable (ε, δ)-differential privacy guarantees** to satisfy ODPC regulatory expectations;
>
> **design**, **implement**, and **validate** a privacy-preserving federated Graph Neural Network framework that:
> 1. Trains a shared R-GCN / GATConv model without any raw transaction data leaving any institution's perimeter,
> 2. Provides (ε, δ)-DP guarantees on all transmitted model updates using Top-K Gradient Sparsification,
> 3. Achieves fraud detection performance (AUC-ROC > 0.95, F1 > 0.85 under class imbalance) superior to institution-siloed baselines,
> 4. Operates strictly within the 50ms inference latency boundary (specifically aiming for a mathematically proven 16.6ms pipeline) using FP16 mixed-precision model weights inside hardware TEE enclaves (Intel TDX / AMD SEV-SNP),
> 5. Remains legally compliant with the Kenya Data Protection Act 2019, CBK Prudential Guidelines on Risk Management, and FATF AML/CFT enhanced supervision requirements.

---

## 9. Conclusion

Kenya's interbank payment infrastructure represents one of the most sophisticated and high-velocity real-time settlement environments on the continent. The PesaLink network, connecting 80+ financial institutions through multiple digital access channels, processes transactions that can be formally represented as a directed, temporal, heterogeneous graph, a mathematical structure that encodes rich fraud-detecting signals invisible to conventional tabular analysis. The CBK's documented fraud escalation (losses up to KES 1.90 billion in 2024, with a +264% net loss increase year-on-year), combined with the High Court's landmark liability ruling and the ODPC's stringent data minimisation requirements, creates a tripartite mandate that no existing architecture fully satisfies. This work addresses that gap through the design of a privacy-preserving federated graph learning framework, one that respects institutional data sovereignty while enabling the collective, cross-bank fraud intelligence that the sophistication of modern criminal syndicates demands.

---

## References

[1] Integrated Payment Services Limited (IPSL). *PesaLink: About Us*. https://pesalink.co.ke/about-us (Accessed July 2026).

[2] Central Bank of Kenya. *Bank Supervision Annual Report 2024 / Financial Sector Stability Report 2024*. Nairobi: Central Bank of Kenya. https://www.centralbank.go.ke/reports/bank-supervision-and-banking-sector-reports/

[3] Tech-Ish. (2026, July 13). *Court Rules a Correct PIN Is Not a Defence: Safaricom and DTB to Pay KES 4.4M SIM Swap Victim*. https://tech-ish.com/2026/07/13/safaricom-dtb-sim-swap-ruling/

[4] Central Bank of Kenya. *National Payments System*. https://www.centralbank.go.ke/national-payments-system/ (Accessed July 2026).

[5] Africa Nenda Foundation. (2022). *The State of Instant and Inclusive Payment Systems in Africa: PesaLink Case Study*. https://africanenda.org/wp-content/uploads/EN_SIIPS_Casestudy_Pesalink.pdf

[6] Wang, X., et al. (2021). *The Banking Transactions Dataset and its Comparative Analysis with Scale-free Networks*. ResearchGate. https://www.researchgate.net/publication/354778270_The_Banking_Transactions_Dataset_and_its_Comparative_Analysis_with_Scale-free_Networks

[7] National Council for Law Reporting. (2019). *The Data Protection Act, No. 24 of 2019*. Nairobi: Government Printer of Kenya.

[8] Lin, T. Y., Goyal, P., Girshick, R., He, K., & Dollár, P. (2017). Focal loss for dense object detection. *Proceedings of the IEEE International Conference on Computer Vision (ICCV)*, 2980–2988.

[9] Schlichtkrull, M., Kipf, T. N., Bloem, P., van den Berg, R., Titov, I., & Welling, M. (2018). Modeling relational data with graph convolutional networks. *European Semantic Web Conference (ESWC)*, 593–607.

[10] Veličković, P., Cucurull, G., Casanova, A., Romero, A., Liò, P., & Bengio, Y. (2018). Graph attention networks. *International Conference on Learning Representations (ICLR)*. https://arxiv.org/abs/1710.10903

[11] Chiang, W. L., Liu, X., Si, S., Li, Y., Bengio, S., & Hsieh, C. J. (2019). Cluster-GCN: An efficient algorithm for training deep and large graph convolutional networks. *Proceedings of the 25th ACM SIGKDD*, 257–266. https://arxiv.org/abs/1905.07953

[12] McMahan, B., Moore, E., Ramage, D., Hampson, S., & Arcas, B. A. (2017). Communication-efficient learning of deep networks from decentralized data. *Proceedings of the 20th International Conference on Artificial Intelligence and Statistics (AISTATS)*, PMLR 54:1273–1282. https://proceedings.mlr.press/v54/mcmahan17a.html

[13] Zhao, B., Mopuri, K. R., & Bilen, H. (2020). idLG: Improved Deep Leakage from Gradients. *arXiv:2001.02610*.

[14] Abadi, M., Chu, A., Goodfellow, I., McMahan, H. B., Mironov, I., Talwar, K., & Zhang, L. (2016). Deep learning with differential privacy. *Proceedings of the 2016 ACM SIGSAC CCS*, 308–318.

[15] Bonawitz, K., Ivanov, V., Kreuter, B., Marcedone, A., McMahan, H. B., Patel, S., Ramage, D., Segal, A., & Seth, K. (2017). Practical secure aggregation for privacy-preserving machine learning. *Proceedings of the 2017 ACM SIGSAC CCS*, 1175–1191.

[16] Subramanyan, P., Sinha, S., Lebedev, I., Devadas, S., & Seshia, S. A. (2017). A formal foundation for secure remote attestation. *Proceedings of the 2017 ACM SIGSAC CCS*, 2435–2449.

[17] Hamilton, W. L., Ying, R., & Leskovec, J. (2017). Inductive representation learning on large graphs. *Advances in Neural Information Processing Systems (NeurIPS)*, 30. https://arxiv.org/abs/1706.02216

[18] Financial Action Task Force (FATF). (2024). *Kenya Mutual Evaluation Follow-up Report*. Paris: FATF. https://www.fatf-gafi.org/en/countries/detail/Kenya.html

[19] Office of the Data Protection Commissioner (ODPC). *Data Protection Act, 2019: Regulatory Guidance*. Nairobi: Government of Kenya. https://odpc.go.ke

[20] Tahir, M., et al. (2024). Federated learning models for privacy-preserving fraud analytics across global banking networks. *ResearchGate preprint*. https://www.researchgate.net/publication/397779653_Federated_Learning_Models_for_Privacy-Preserving_Fraud_Analytics_Across_Global_Banking_Networks
[21] Ben-Sasson, E., Chiesa, A., Genkin, D., Tromer, E., & Virza, M. (2013). SNARKs for C: Verifying program executions succinctly and in zero knowledge. *Advances in Cryptology, CRYPTO 2013*, LNCS 8043. https://eprint.iacr.org/2013/507
[22] Kocsis, L., & Szepesvári, C. (2006). Bandit based Monte-Carlo planning. *European Conference on Machine Learning (ECML 2006)*. Lecture Notes in Computer Science, vol. 4212, pp. 282–293. Springer, Berlin. https://link.springer.com/chapter/10.1007/11871842_29
[23] Ghorbani, A., & Zou, J. (2019). Data Shapley: Equitable valuation of data for machine learning. *Proceedings of the 36th International Conference on Machine Learning (ICML 2019)*, PMLR 97:2242–2251. https://arxiv.org/abs/1904.02868
[24] Xu, J., Glicksberg, B. S., Su, C., Walker, P., Bian, J., & Wang, F. (2021). Federated learning for healthcare informatics. *Journal of Healthcare Informatics Research*, 5, 1–19; and: Song, T., Tong, Y., & Wei, S. (2022). GTG-Shapley: Efficient and accurate participant contribution evaluation in federated learning. *ACM Transactions on Intelligent Systems and Technology (TIST)*, 13(4), Article 60. https://arxiv.org/abs/2109.02053
[25] Saxena, A., et al. (2021). *The Banking Transactions Dataset and its Comparative Analysis with Scale-free Networks*. ResearchGate.
[26] Rishabh, K. (2026). *Beyond the bureau: Interoperable payment data for loan screening and monitoring*. ScienceDirect.
[27] Wang, H., et al. (2020). *A Bipartite Graph-Based Recommender for Crowdfunding with Sparse Data*. ResearchGate.
