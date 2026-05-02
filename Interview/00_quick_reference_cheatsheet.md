# PwC Blockchain Interview — Quick Reference Cheat Sheet
## Read this the morning of your interview

---

## TOP 10 MOST LIKELY TECHNICAL QUESTIONS

| # | Question | 30-Second Answer |
|---|---|---|
| 1 | Hyperledger Fabric architecture | Peers + Orderers + CA + Channels + Chaincode + MSP |
| 2 | Fabric transaction flow | Client → Endorsing Peers → Orderer → All Peers commit |
| 3 | Public vs Permissioned blockchain | Open/anyone vs Role-based access, different consensus |
| 4 | Reentrancy attack | Call external before updating state = vulnerable. Fix: update state first (CEI pattern) |
| 5 | ERC-20 vs ERC-721 | Fungible token vs NFT (unique tokenId) |
| 6 | IBFT vs Raft | IBFT=BFT(tolerates malicious), Raft=CFT(crash only) |
| 7 | Channel vs Private Data Collection | Channel=separate ledger, PDC=shared channel private side-DB |
| 8 | How to deploy chaincode Fabric 2.x | Package→Install→ApproveForMyOrg→Commit |
| 9 | delegatecall | Runs external code in YOUR context/storage (proxy pattern) |
| 10 | Docker vs Kubernetes | Compose=dev single host, K8s=prod multi-host auto-scaling |

---

## HYPERLEDGER PLATFORMS QUICK COMPARE

| Platform | Language | EVM? | Use Case | Consensus |
|---|---|---|---|---|
| **Fabric** | Go/Node/Java chaincode | No | Supply chain, trade finance | Raft |
| **Besu** | Solidity (EVM) | **Yes** | Private Ethereum network | IBFT2/QBFT |
| **Corda** | Kotlin/Java | No | Finance, insurance | Notary service |
| **Indy** | Python/Rust | No | Identity, SSI | Plenum BFT |
| **Sawtooth** | Multi-language | No | IoT, supply chain | PoET (Intel SGX) |

---

## SOLIDITY VULNERABILITY QUICK LIST

```
1. Reentrancy       → update state BEFORE external call (CEI pattern)
2. tx.origin auth   → use msg.sender, not tx.origin  
3. Integer overflow → use Solidity 0.8+ or SafeMath
4. Unchecked .call  → always check (bool success,) = addr.call{value:x}("")
5. Flash loan       → use TWAP/Chainlink, not spot price
6. Front-running    → commit-reveal scheme, slippage limits
7. Access control   → OpenZeppelin Ownable/AccessControl
```

---

## FABRIC KEY TERMS

```
MSP          = Membership Service Provider (identity → org mapping)
Anchor Peer  = Public peer for cross-org gossip discovery
Endorsement  = Peer simulates & signs tx before ordering
Orderer      = Creates/orders blocks (Raft-based)
World State  = Current state DB (CouchDB or LevelDB)
Ledger       = World State + Block log
Channel      = Private subnet with its own ledger
PDC          = Private Data Collection (partial privacy within channel)
CRL          = Certificate Revocation List
```

---

## AWS SERVICES FOR BLOCKCHAIN

```
EKS          → Run Fabric nodes on Kubernetes
ECR          → Store Docker images for chaincode
S3           → Off-chain document storage
Lambda       → Chaincode event processors
SQS/SNS      → Async messaging from blockchain events
RDS          → Off-chain query DB (synced from events)
Secrets Mgr  → Store private keys, TLS certs
CloudWatch   → Monitor node health, tx rates
Managed BC   → Fully managed Fabric service (exists, limited)
```

---

## TALKING POINTS — YOUR PROJECTS FOR PwC

### Peer(VINE) Chain → "Foundation"
Node setup, genesis config, NPOS consensus, Substrate basics

### 5IRE Chain → "Innovation"
EVM+WASM dual compatibility (Frontier), ESG sustainability pallet — **sustainability aligns with PwC values**

### Saita Chain → "Scale"
Similar EVM work, showed I can repeat and improve process

### CTRL ALT (Algorand) → "Adaptability"
Learned new platform quickly, delivered Voting contract — shows I can onboard to Fabric fast

### HYDRO (Sui Move) → "DeFi Expertise"
Token, Vesting, Staking, Rewards Distribution — full DeFi contract suite — relevant to PwC tokenization work

---

## BLOCKCHAIN DECISION FRAMEWORK (For Case Study)

```
Q: Should client use blockchain?
├── Multiple parties who don't trust each other? 
│   ├── YES → Consider blockchain
│   └── NO  → Use regular database
├── Need shared, immutable record?
│   ├── YES → Blockchain adds value
│   └── NO  → Overkill
└── Which platform?
    ├── Need Ethereum tooling / public compatibility → Besu
    ├── Need fine-grained access control, complex workflows → Fabric
    ├── Financial contracts, legal integration → Corda
    └── Identity / SSI use case → Indy
```

---

## NUMBERS TO REMEMBER

```
Fabric block time        ~0.5-2 seconds (configurable)
Fabric tx throughput     ~3,000+ TPS (benchmark)
Ethereum block time      ~12 seconds
Ethereum mainnet TPS     ~15-30 TPS
Polygon PoS TPS          ~7,000 TPS
PBFT requirement         3f+1 nodes for f faults
BFT min validators       4 (tolerates 1 faulty)
ERC-20 functions         6 core functions
Gas for simple tx        21,000 gas units
```

---

## INTERVIEW DAY CHECKLIST

- [ ] Review Fabric transaction flow one more time
- [ ] Know your ESG pallet story cold — PwC loves sustainability angle
- [ ] Prepare: "Why PwC?" (consulting + diversity of clients + scale)
- [ ] Prepare: "Bridge the Fabric gap" answer (conceptual overlap + quick learner)
- [ ] Questions to ask interviewer (see Part 4)
- [ ] Know your HYDRO project numbers (how many contracts, what kind)
- [ ] Be ready to whiteboard a supply chain solution on Fabric
- [ ] Dress: Business formal (PwC = Big 4, professional environment)

---

## COMMON PwC INTERVIEW MISTAKES TO AVOID

1. **Saying "I don't know Fabric"** → Always bridge: "I haven't used it in production, but I've studied it extensively and it parallels my Substrate experience in X, Y ways"
2. **Too technical too fast** → PwC values clear communication. Structure your answers
3. **Not asking questions** → Shows low engagement. Use the questions from Part 4
4. **Underselling your Rust experience** → It shows deep systems understanding
5. **Not connecting to business value** → Always tie technical choices to business outcomes

---

## FILES IN THIS FOLDER

| File | Content |
|---|---|
| `00_quick_reference_cheatsheet.md` | This file — read before interview |
| `01_core_blockchain_hyperledger.md` | Fabric, Besu, Corda, Indy, Sawtooth (25 Q&A) |
| `02_ethereum_solidity_smart_contracts.md` | Ethereum, Solidity, ERC standards, security (25 Q&A) |
| `03_system_design_devops_cloud.md` | Architecture, Docker, K8s, AWS, Databases, APIs (21 Q&A) |
| `04_hr_behavioral_background.md` | HR, behavioral, resume-specific, PwC-specific (17 Q&A) |

**Total: 88 questions with detailed answers**

---

## GOOD LUCK!

Your background is stronger than it seems for this role:
- Substrate/Rust = deep consensus + systems knowledge (rare)
- Sui Move = modern smart contract security patterns
- Algorand = proof of adaptability
- 5IRE ESG = sustainability angle (PwC loves this)
- 3 years production blockchain = real-world scars, not just theory

The gap: Fabric/Corda direct experience. **Own it, bridge it confidently.**
