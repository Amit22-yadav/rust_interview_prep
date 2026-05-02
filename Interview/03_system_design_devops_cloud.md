# PwC Senior Associate - Blockchain Developer Interview Prep
## Part 3: System Design, DevOps, Cloud & Databases

---

## SYSTEM DESIGN FOR BLOCKCHAIN

### Q1. How would you design a trade finance platform on blockchain?
**Answer (Structure your answer as: Problem → Solution → Architecture):**

**Problem**: Multiple parties (Buyer, Seller, Bank, Logistics, Customs) don't trust each other's data.

**Solution**: Permissioned blockchain (Hyperledger Fabric) for shared ledger.

**Architecture**:
```
Organizations: BuyerBank, SellerBank, LogisticsProvider, CustomsAuthority

Channels:
- main-channel (all orgs): Trade lifecycle events
- bank-channel (BuyerBank + SellerBank): LC (Letter of Credit) details
- logistics-channel (Seller + Logistics): Shipment data

Chaincode:
- TradeContract: createTrade(), submitLC(), confirmShipment(), releaseFunds()
- DocumentContract: submitBOL(), verifyDocument()

Off-chain:
- Documents (PDF): IPFS → hash on-chain
- REST API: Node.js + Fabric SDK
- Database: PostgreSQL for query-optimized views (event-driven sync)
- Frontend: React.js

Privacy:
- Private Data Collections: Bank-to-bank pricing terms
```

---

### Q2. How do you handle scalability in enterprise blockchain networks?
**Answer:**
- **Horizontal scaling**: Add more peers per org
- **Channel separation**: High-volume data on dedicated channels
- **Off-chain compute**: Move heavy computation off-chain, store results on-chain
- **Event-driven architecture**: Listen to chaincode events, sync to SQL DB for complex queries
- **Caching**: Redis cache for frequently read world state
- **Batch processing**: Group multiple txs in one chaincode invocation
- **Fabric orderer tuning**: `BatchSize`, `BatchTimeout` parameters

---

### Q3. How would you design a KYC/AML system on Hyperledger Indy?
**Answer:**
- **Issuer** (Bank/Government): Issues verifiable credentials (ID, address proof)
- **Holder** (Customer): Stores credentials in digital wallet (Aries agent)
- **Verifier** (PwC Client): Requests proof, verifies without seeing raw data

**Zero-Knowledge Proof**: Customer proves "age > 18" without revealing actual birth date.

**Benefits for PwC clients**:
- No central storage of PII
- Customer controls their data
- Audit trail on ledger
- Cross-institution identity sharing

---

### Q4. How do you integrate blockchain with existing enterprise systems (ERP, CRM)?
**Answer:**
- **Event listener service**: Monitors blockchain events → triggers ERP updates
- **REST API gateway**: Wraps blockchain SDK calls
- **Message queue** (Kafka/RabbitMQ): Async integration layer
- **Middleware**: Enterprise Service Bus (ESB) pattern
- SAP has **SAP Blockchain** connector
- IBM has **Blockchain Platform** with API Connect

```
ERP → REST API → Node.js Service → Fabric SDK → Blockchain
           ↑
       Events via Kafka
```

---

### Q5. What is the CAP theorem and how does it apply to blockchain?
**Answer:**
- **Consistency**: All nodes see same data
- **Availability**: Every request gets a response
- **Partition Tolerance**: System works despite network splits

Blockchain networks generally choose **Consistency + Partition Tolerance** (CP):
- Bitcoin: Eventually consistent (probabilistic finality after 6 blocks)
- Hyperledger Fabric: Strong consistency (PBFT/Raft gives deterministic finality)
- Ethereum PoS: Probabilistic finality → moving toward deterministic (Casper FFG)

---

## DOCKER & KUBERNETES

### Q6. How is Docker used in Hyperledger Fabric?
**Answer:**
- Each peer, orderer, CA runs in its own **Docker container**
- Chaincode (Go, Node.js) runs in a **separate Docker container** per channel per chaincode
- `docker-compose.yml` defines the full network topology
- Volumes for persistent ledger data
- Networks for org isolation

```yaml
# Sample peer service in docker-compose
peer0.org1.example.com:
  image: hyperledger/fabric-peer
  environment:
    - CORE_PEER_ID=peer0.org1.example.com
    - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
    - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
  volumes:
    - /var/run/docker.sock:/host/var/run/docker.sock
    - ./crypto-config:/etc/hyperledger/fabric/crypto-config
```

---

### Q7. How would you deploy a blockchain network on Kubernetes?
**Answer:**
- Each Fabric component → Kubernetes **Deployment** + **Service**
- Persistent state → **PersistentVolumeClaim** (for ledger data)
- Crypto material → **Kubernetes Secrets**
- Config → **ConfigMaps**
- Cross-org networking → **Ingress** or **LoadBalancer**
- Tools: **Helm charts** for Fabric (hlf-operator, MinFabric)
- **Hyperledger Bevel**: Ansible-based automation for Fabric on K8s

Key challenges:
- Dynamic service discovery across orgs
- TLS certificate management
- Persistent storage for ledger
- Rolling upgrades for peers

---

### Q8. Explain the difference between Docker Compose and Kubernetes.
**Answer:**
| | Docker Compose | Kubernetes |
|---|---|---|
| Use case | Dev/test single host | Production multi-host |
| Scaling | Manual | Auto-scaling |
| Self-healing | No | Yes (restarts failed pods) |
| Load balancing | No | Built-in |
| Config management | .env files | ConfigMaps, Secrets |
| Complexity | Simple | Complex |

For PwC: Dev/test = Docker Compose. Production = Kubernetes on AWS EKS or Azure AKS.

---

## CLOUD PLATFORMS

### Q9. How do you deploy Hyperledger Fabric on AWS?
**Answer:**
**AWS Managed Blockchain** (fully managed):
- Supports Hyperledger Fabric and Ethereum
- Auto-manages peer nodes, orderers, CA
- VPC integration, CloudWatch monitoring
- `aws managedblockchain create-network`

**Self-managed on AWS EKS**:
- EKS for container orchestration
- EFS for persistent shared storage (ledger)
- ACM for TLS certificates
- ALB for API gateway
- CloudWatch for logs/metrics
- S3 for chaincode packaging

---

### Q10. What AWS services are used in blockchain applications?
**Answer:**
- **EC2/EKS**: Run blockchain nodes
- **S3**: Store chaincode packages, config files, off-chain documents
- **RDS/DynamoDB**: Store synced blockchain state for queries
- **Lambda**: Event-driven processors (chaincode event listeners)
- **SNS/SQS**: Async messaging for blockchain events
- **API Gateway**: REST API for blockchain interaction
- **Secrets Manager**: Store private keys, TLS certificates
- **CloudWatch**: Monitor node health, transaction rates
- **VPC**: Network isolation for private networks

---

### Q11. How does Azure Blockchain Service differ from AWS Managed Blockchain?
**Answer:**
Azure Blockchain Service was **retired in Sep 2021** — now Microsoft recommends:
- **ConsenSys Quorum** on Azure
- **Azure Kubernetes Service (AKS)** for self-managed
- **Azure Confidential Ledger**: Tamper-proof ledger using TEE

AWS Managed Blockchain is still active and growing.

For PwC interviews, focus on practical deployment on **EKS** and knowing **AWS Managed Blockchain** exists.

---

## DATABASES

### Q12. When would you use CouchDB vs PostgreSQL in a blockchain project?
**Answer:**
- **CouchDB** (Fabric World State): For real-time on-chain state queries, rich JSON queries within chaincode
- **PostgreSQL** (off-chain): For complex joins, historical analytics, BI reporting

**Pattern**: 
1. Listen to chaincode events
2. Parse events in event listener service
3. Sync to PostgreSQL for reporting
4. Business intelligence runs against PostgreSQL

---

### Q13. What is the difference between MongoDB and PostgreSQL for blockchain backends?
**Answer:**
| | MongoDB | PostgreSQL |
|---|---|---|
| Schema | Flexible (document) | Strict (relational) |
| Queries | Aggregation pipeline | Full SQL |
| Joins | Limited ($lookup) | Native JOIN |
| ACID | Multi-doc tx (v4+) | Full ACID |
| Blockchain use | Store raw events, flexible docs | Normalized state, reporting |

Hyperledger Fabric's CouchDB uses JSON like MongoDB — familiar for Fabric devs.

---

### Q14. How do you handle database schema migration in a blockchain project?
**Answer:**
- Off-chain DB schema: Use **Flyway** or **Liquibase** for versioned migrations
- On-chain state schema: Add version field to chaincode state
  ```go
  type Asset struct {
      DocType string `json:"docType"` // for CouchDB query
      Version int    `json:"version"`
      // ... fields
  }
  ```
- Handle migration in chaincode: Check version, upgrade state on first access
- Never delete old fields — mark as deprecated

---

## REST APIs & WEB SERVICES

### Q15. How do you design a REST API for a blockchain application?
**Answer:**
```
POST   /api/assets              → invoke chaincode (create)
GET    /api/assets/:id          → query chaincode
PUT    /api/assets/:id          → invoke chaincode (update)
GET    /api/assets/:id/history  → chaincode GetHistoryForKey
GET    /api/assets?owner=Alice  → rich query (CouchDB)

POST   /api/auth/enroll         → Fabric CA enrollment
POST   /api/auth/register       → Register new user
```

**Authentication**: JWT tokens + Fabric identity mapping
**Rate limiting**: Express-rate-limit
**Error handling**: Standardized error responses
**Async patterns**: Long-running blockchain txs → polling endpoint or WebSocket

---

### Q16. How do you handle async transactions in blockchain REST APIs?
**Answer:**
Blockchain txs take time (endorsement → ordering → commit = 2-5 seconds in Fabric).

**Option 1: Synchronous** (wait for commit):
```javascript
const result = await contract.submitTransaction('CreateAsset', ...args);
res.json({ txId: result.transactionId, data: result });
```

**Option 2: Async** (return immediately):
```javascript
const txId = await contract.createTransaction('CreateAsset').submit();
res.json({ txId, status: 'pending' });
// Client polls GET /transactions/:txId for status
```

**Option 3: WebSocket**: Push commit events to client via WebSocket.

---

### Q17. How do you secure a blockchain REST API?
**Answer:**
- **JWT authentication**: Verify identity before calling chaincode
- **Role-based access control (RBAC)**: Map JWT roles to allowed chaincode functions
- **TLS/HTTPS**: Encrypt all API traffic
- **Input validation**: Validate and sanitize all inputs (prevent injection)
- **Rate limiting**: Prevent DoS
- **API key management**: For service-to-service communication
- **Fabric identity**: Each API user should have their own Fabric cert (not shared admin cert)
- **Audit logging**: Log all API calls with user identity and tx ID

---

## LINUX & SCRIPTING

### Q18. What Linux commands are essential for blockchain operations?
**Answer:**
```bash
# Docker operations
docker ps -a                          # list all containers
docker logs peer0.org1 --tail 100    # tail peer logs
docker exec -it peer0.org1 bash       # shell into container
docker stats                          # resource usage

# File/cert operations
openssl x509 -in cert.pem -text      # inspect certificate
openssl verify -CAfile ca.crt cert.pem

# Network
netstat -tlnp | grep 7051            # check port
curl -s http://localhost:7054/cainfo  # check CA

# Process management
systemctl status fabric-peer          # service status
journalctl -u fabric-peer -f          # follow logs
```

---

### Q19. Write a bash script to start a Fabric network and deploy chaincode.
**Answer:**
```bash
#!/bin/bash
set -e  # exit on error

CHANNEL_NAME="mychannel"
CC_NAME="basic"
CC_PATH="./chaincode/go"

echo "=== Starting Fabric Network ==="
docker-compose -f docker-compose.yml up -d

echo "=== Creating Channel ==="
peer channel create -o orderer.example.com:7050 \
  -c $CHANNEL_NAME \
  -f ./channel-artifacts/channel.tx \
  --tls --cafile $ORDERER_CA

echo "=== Joining Channel ==="
peer channel join -b ${CHANNEL_NAME}.block

echo "=== Deploying Chaincode ==="
peer lifecycle chaincode package ${CC_NAME}.tar.gz \
  --path $CC_PATH --lang golang --label ${CC_NAME}_1.0

peer lifecycle chaincode install ${CC_NAME}.tar.gz

PACKAGE_ID=$(peer lifecycle chaincode queryinstalled | grep ${CC_NAME} | awk '{print $3}' | tr -d ',')

peer lifecycle chaincode approveformyorg \
  --channelID $CHANNEL_NAME --name $CC_NAME \
  --version 1.0 --package-id $PACKAGE_ID --sequence 1

peer lifecycle chaincode commit \
  --channelID $CHANNEL_NAME --name $CC_NAME \
  --version 1.0 --sequence 1

echo "=== Chaincode deployed successfully ==="
```

---

### Q20. What are SSL/TLS certificates and how are they used in Fabric?
**Answer:**
- **TLS certificates**: Encrypt peer-to-peer and client-to-peer communication
- **Signing certificates**: Sign transactions (identity)
- In Fabric, each node has:
  - `tls/server.crt` — TLS cert for incoming connections
  - `tls/ca.crt` — TLS CA cert for verifying other nodes
  - `msp/signcerts/` — Identity cert for signing txs
  - `msp/keystore/` — Private key

- Cert expiry causes network outage → monitor with `openssl x509 -enddate`
- Renewal: Re-enroll with Fabric CA, distribute new certs, restart nodes

---

## CICD & DEVOPS

### Q21. How would you set up CI/CD for a blockchain project?
**Answer:**
```
GitHub Push → GitHub Actions / Jenkins Pipeline:
  1. Lint & unit test chaincode (go test ./...)
  2. Slither static analysis on Solidity contracts
  3. Build Docker images
  4. Deploy to test Fabric network (docker-compose)
  5. Run integration tests
  6. Build and push images to ECR/Docker Hub
  7. Deploy to staging K8s cluster (kubectl apply / helm upgrade)
  8. Run E2E tests
  9. Manual approval gate
  10. Deploy to production
```

Key tools:
- **GitHub Actions / Jenkins / GitLab CI**: Pipeline orchestration
- **Docker**: Build images
- **Helm**: Package Kubernetes deployments
- **Hardhat CI**: Smart contract test runner
- **Slither**: Security analysis in CI
