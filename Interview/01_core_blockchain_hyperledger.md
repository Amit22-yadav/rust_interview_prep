# PwC Senior Associate - Blockchain Developer Interview Prep
## Part 1: Core Blockchain & Hyperledger (Most Expected)

---

## BLOCKCHAIN FUNDAMENTALS

### Q1. What is the difference between public, private, and permissioned blockchains?
**Answer:**
| Feature | Public | Private | Permissioned |
|---|---|---|---|
| Access | Anyone | Restricted | Role-based |
| Consensus | PoW/PoS | BFT/Raft | PBFT/Raft |
| Examples | Bitcoin, Ethereum | Hyperledger Fabric | R3 Corda, Quorum |
| Speed | Slow | Fast | Fast |
| Use Case | Crypto | Internal corp | Enterprise B2B |

PwC deals with **enterprise/permissioned** blockchains primarily.

---

### Q2. What is Byzantine Fault Tolerance (BFT)?
**Answer:** A system's ability to continue operating correctly even if some nodes fail or act maliciously. A system tolerates up to `f` faulty nodes if it has at least `3f+1` total nodes.

- **PBFT** (Practical BFT): Used in Hyperledger Fabric, requires known participants
- **Raft**: Simpler, CFT (crash fault tolerant) only — not BFT
- Ethereum uses Casper (PoS-based finality)

---

### Q3. Explain consensus mechanisms used in enterprise blockchains.
**Answer:**
- **Raft** (Hyperledger Fabric 2.x ordering service): Leader-based, crash fault tolerant, fast
- **PBFT**: Used in older Fabric versions (Kafka-based ordering)
- **Istanbul BFT (IBFT)**: Used in Hyperledger Besu, Quorum — BFT variant
- **Clique PoA**: Proof of Authority, used in Besu for private networks
- **QBFT**: Quorum BFT — improved IBFT used in Besu

---

### Q4. What is Hyperledger Fabric architecture?
**Answer:** Key components:
- **Peer**: Hosts ledger and chaincode. Types: Endorsing peer, Committing peer, Anchor peer
- **Orderer**: Creates ordered blocks (Raft-based). Ensures consistent transaction ordering
- **CA (Certificate Authority)**: Issues X.509 certs for identity (Fabric CA)
- **Channel**: Private subnet between org subsets — data isolation
- **MSP (Membership Service Provider)**: Maps certs to org identities
- **Chaincode (Smart Contract)**: Business logic, runs in Docker containers
- **Ledger**: World State (CouchDB/LevelDB) + Blockchain (transaction log)

**Transaction flow:**
1. Client proposes tx → Endorsing peers simulate & sign
2. Client collects endorsements → Sends to Orderer
3. Orderer batches → creates block → distributes to all peers
4. Peers validate & commit to ledger

---

### Q5. What is the difference between LevelDB and CouchDB in Hyperledger Fabric?
**Answer:**
| | LevelDB | CouchDB |
|---|---|---|
| Type | Key-Value store | Document store (JSON) |
| Query | Key-only lookup | Rich JSON queries |
| Performance | Faster | Slower (network call) |
| Use Case | Simple state | Complex queries needed |

Use **CouchDB** when you need to query state by fields other than key (e.g., "get all assets owned by Alice").

---

### Q6. What are Fabric channels and why are they used?
**Answer:** Channels provide **data isolation** between org subsets on the same network. Each channel has its own:
- Ledger
- Chaincode
- Membership

Example: In a supply chain network, Org A and Org B may have a private channel for pricing data that Org C cannot see.

---

### Q7. Explain chaincode lifecycle in Hyperledger Fabric 2.x.
**Answer:** New decentralized lifecycle (Fabric 2.0+):
1. `package` — Developer packages chaincode
2. `install` — Each org installs on their peers
3. `approveForMyOrg` — Each org approves the chaincode definition
4. `commit` — Once enough orgs approve (per endorsement policy), chaincode is committed to channel
5. `init` (optional) — Initialize chaincode state

This ensures **no single org can unilaterally upgrade** chaincode.

---

### Q8. What is an endorsement policy?
**Answer:** Defines which peers must endorse (simulate + sign) a transaction for it to be valid.

Examples:
- `AND('Org1.peer', 'Org2.peer')` — both must endorse
- `OR('Org1.peer', 'Org2.peer')` — either can endorse
- `OutOf(2, 'Org1.peer', 'Org2.peer', 'Org3.peer')` — any 2 of 3

---

### Q9. What is Hyperledger Besu and how does it differ from Fabric?
**Answer:**
| | Hyperledger Besu | Hyperledger Fabric |
|---|---|---|
| Language | Java | Go |
| EVM | Yes (Ethereum-compatible) | No |
| Smart Contracts | Solidity | Go, Node.js, Java |
| Consensus | IBFT2, QBFT, Clique, PoA | Raft, PBFT |
| Privacy | Tessera (private tx) | Channels, Private Data Collections |
| Use Case | EVM-compatible private networks | Supply chain, trade finance |

Besu is used when you want **Ethereum tooling** (Truffle, Hardhat, MetaMask) in a private network.

---

### Q10. What are Private Data Collections in Hyperledger Fabric?
**Answer:** Allows a **subset of orgs** within the same channel to keep data private — more granular than channels.

- Private data stored in a **side database** (private state DB)
- Only hash of private data goes to the main ledger
- Other orgs can verify the hash but not see the data
- Useful when you don't want to create a new channel just for partial privacy

---

### Q11. What is R3 Corda and when would you use it?
**Answer:** Corda is a DLT platform designed for **financial institutions**.

Key differences from Fabric:
- **No global ledger** — only parties to a transaction share data (point-to-point)
- **States, Contracts, Flows** instead of chaincode
- Built on JVM (Kotlin/Java)
- Strong legal prose integration (Ricardian contracts)
- Use cases: Trade finance, securities settlement, insurance

---

### Q12. What is Hyperledger Indy and what problem does it solve?
**Answer:** Indy is a blockchain for **Self-Sovereign Identity (SSI)**.

- Issues **Decentralized Identifiers (DIDs)**
- Supports **Verifiable Credentials** — users hold their own identity, not stored in a central DB
- Used in Hyperledger Aries (agent framework) for real-world identity
- PwC use case: KYC, AML, digital identity for clients

---

### Q13. Explain Hyperledger Sawtooth and its unique consensus.
**Answer:** Sawtooth uses **Proof of Elapsed Time (PoET)**:
- Uses Intel SGX (trusted hardware) to generate random wait times
- Node that waits least gets to create the block
- Energy efficient alternative to PoW
- Also supports PBFT

Key feature: **Transaction Families** — modular smart contract system. Built-in families: IntegerKey, Settings, Identity.

---

### Q14. What is the Multichain blockchain platform?
**Answer:** MultiChain is a private blockchain platform based on Bitcoin protocol.
- Permission-based access control
- Multi-asset support natively
- Streams for key-value data storage
- Used for simple private ledger use cases
- No smart contract support (unlike Ethereum/Fabric)

---

### Q15. How do you set up a Hyperledger Fabric network from scratch?
**Answer:** High-level steps:
1. **Prerequisites**: Docker, Docker Compose, Go, Node.js, Fabric binaries
2. **Crypto material**: `cryptogen` tool or Fabric CA to generate certs
3. **Genesis block**: `configtxgen` to create orderer genesis block
4. **Channel config**: Create channel tx using `configtxgen`
5. **Start network**: Docker Compose with peer, orderer, CA containers
6. **Create channel**: `peer channel create`
7. **Join channel**: `peer channel join`
8. **Deploy chaincode**: Package → Install → Approve → Commit
9. **Invoke/Query**: `peer chaincode invoke/query`

---

### Q16. What is MSP (Membership Service Provider)?
**Answer:** MSP defines the rules for validating identities in Fabric:
- Maps X.509 certificates to organizational roles
- Every org has its own MSP
- Contains: Root CA cert, Admin certs, Revocation lists (CRL)
- Types: Local MSP (for a node) vs Channel MSP (network-wide)

---

### Q17. What are anchor peers in Fabric?
**Answer:** Anchor peers are **publicly known peers** used for **cross-org gossip communication**.

- Each org designates at least one anchor peer per channel
- Other orgs discover peers through anchor peers
- Configured in channel configuration
- Critical for service discovery to work across orgs

---

### Q18. What is Gossip Protocol in Hyperledger Fabric?
**Answer:** Used for peer-to-peer data dissemination:
- Block distribution from orderer to peers
- State synchronization between peers
- Peer discovery across orgs (via anchor peers)
- Reduces load on orderer — only one peer gets block, gossips to others

---

### Q19. How does identity work in Fabric? Explain X.509.
**Answer:**
- Every participant (user, peer, orderer) has an X.509 certificate
- Issued by Fabric CA or external CA
- Certificate contains: Subject, Issuer, Public Key, Validity, Extensions
- MSP uses these certs to determine permissions
- TLS certs (for transport) are separate from signing certs (for tx)

---

### Q20. What are the key differences between Hyperledger Fabric v1.x and v2.x?
**Answer:**
| Feature | v1.x | v2.x |
|---|---|---|
| Chaincode Lifecycle | Instantiate (single admin) | Decentralized (org approval) |
| Ordering Service | Kafka or Solo | Raft |
| Chaincode as External Service | No | Yes |
| State-based Endorsement | No | Yes |
| Token SDK | No | Yes (experimental) |
| Go module support | No | Yes |

---

## PRACTICAL / SCENARIO QUESTIONS

### Q21. How would you design a supply chain solution on Hyperledger Fabric?
**Answer (talking points):**
- Orgs: Manufacturer, Distributor, Retailer, Regulator
- Channels: Main channel (all orgs) + private channels for pricing
- Chaincode: Asset creation, transfer, provenance tracking
- Private Data Collections: Pricing between Manufacturer-Distributor only
- Events: Chaincode events for real-time notifications
- Off-chain storage: IPFS for large documents, hash on-chain

---

### Q22. How do you handle chaincode upgrades without downtime?
**Answer:**
- Fabric 2.x: New decentralized lifecycle — deploy new version, get approvals
- Use versioning in chaincode name or sequence number
- Migration strategy: Add new fields to state, handle nil checks for old data
- Blue-green deployment pattern for zero downtime
- Test on staging channel first

---

### Q23. How do you query blockchain history for an asset?
**Answer:**
```go
// In Fabric chaincode (Go)
func (s *SmartContract) GetHistory(ctx contractapi.TransactionContextInterface, assetID string) ([]HistoryQueryResult, error) {
    resultsIterator, err := ctx.GetStub().GetHistoryForKey(assetID)
    defer resultsIterator.Close()
    
    var records []HistoryQueryResult
    for resultsIterator.HasNext() {
        response, _ := resultsIterator.Next()
        record := HistoryQueryResult{
            TxID:      response.TxId,
            Timestamp: response.Timestamp,
            Value:     string(response.Value),
            IsDelete:  response.IsDelete,
        }
        records = append(records, record)
    }
    return records
}
```

---

### Q24. What is state-based endorsement (SBE) in Fabric 2.x?
**Answer:** Allows setting a **per-key endorsement policy** instead of chaincode-level policy.

- Individual asset keys can have different endorsement requirements
- Useful when asset ownership changes (new owner's org must endorse)
- Set using `SetStateValidationParameter` in chaincode

---

### Q25. How do you expose chaincode functionality via REST API?
**Answer:**
- Use **Fabric SDK** (Node.js, Go, Java) in a backend service
- Node.js: `fabric-network` package
- Flow: REST endpoint → SDK → Gateway → Peer
- Handle wallet/identity management for signing
- Use connection profile (JSON) to configure network topology
- PwC typically uses Node.js SDK + Express.js for this
