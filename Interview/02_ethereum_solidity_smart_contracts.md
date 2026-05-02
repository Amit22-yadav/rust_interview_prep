# PwC Senior Associate - Blockchain Developer Interview Prep
## Part 2: Ethereum, Solidity & Smart Contracts

---

## ETHEREUM FUNDAMENTALS

### Q1. How does Ethereum differ from Bitcoin?
**Answer:**
| | Bitcoin | Ethereum |
|---|---|---|
| Purpose | Digital currency | Programmable blockchain |
| Smart Contracts | No | Yes (EVM) |
| Consensus | PoW → PoS (The Merge) | PoS (since Sep 2022) |
| Block Time | ~10 min | ~12 sec |
| Language | Script (limited) | Solidity, Vyper |
| Supply | Capped at 21M | No hard cap (deflationary via EIP-1559) |

---

### Q2. What is the EVM (Ethereum Virtual Machine)?
**Answer:**
- Stack-based virtual machine that executes bytecode
- **Deterministic**: Same input always gives same output across all nodes
- **Isolated**: Runs in a sandboxed environment
- **Gas metered**: Every opcode has a gas cost
- Smart contracts compile to **EVM bytecode** (Solidity → bytecode via `solc`)
- Besu and other EVM-compatible chains implement the same spec

---

### Q3. What is Gas in Ethereum?
**Answer:**
- Unit measuring computational work
- **Gas Limit**: Max gas sender is willing to pay
- **Gas Price** (pre-EIP-1559): Wei per gas unit
- **Base Fee** (post-EIP-1559): Protocol-determined, burned
- **Priority Fee (tip)**: Goes to validator
- `Transaction Cost = Gas Used × (Base Fee + Priority Fee)`
- Out-of-gas → tx reverts, but gas already used is NOT refunded

---

### Q4. Explain EIP-1559 and its impact.
**Answer:**
- Introduced in London hard fork (Aug 2021)
- **Base fee**: Algorithmically set, **burned** (removed from supply)
- **Max fee**: User sets max they're willing to pay
- **Priority fee**: Tip to validator = `min(maxFee - baseFee, priorityFee)`
- Makes gas fees more **predictable**
- Makes ETH **deflationary** (base fee burn can exceed issuance)

---

### Q5. What are the types of Ethereum accounts?
**Answer:**
1. **EOA (Externally Owned Account)**:
   - Controlled by private key
   - Can initiate transactions
   - No code

2. **Contract Account**:
   - Controlled by code
   - Cannot initiate txs (only triggered by EOA or other contract)
   - Has storage and code

---

## SOLIDITY

### Q6. Explain Solidity data locations: storage, memory, calldata.
**Answer:**
- **storage**: Persistent on-chain state. Expensive to read/write. Used for state variables
- **memory**: Temporary, exists during function call. Cheaper. Used for local variables
- **calldata**: Read-only, non-modifiable. For external function parameters. Cheapest

```solidity
// Example
function process(uint[] calldata data) external {
    uint[] memory temp = new uint[](data.length); // memory copy
    // state.items = data; // would copy to storage
}
```

---

### Q7. What is the difference between `public`, `private`, `internal`, `external`?
**Answer:**
| Modifier | Inside Contract | Derived Contract | External Call |
|---|---|---|---|
| `public` | ✅ | ✅ | ✅ |
| `private` | ✅ | ❌ | ❌ |
| `internal` | ✅ | ✅ | ❌ |
| `external` | ❌ | ❌ | ✅ |

`external` is gas-efficient for large arrays since calldata is not copied.

---

### Q8. What are common Solidity vulnerabilities?
**Answer:**

**1. Reentrancy Attack**:
```solidity
// VULNERABLE
function withdraw() public {
    uint amount = balances[msg.sender];
    (bool success,) = msg.sender.call{value: amount}(""); // external call BEFORE state update
    balances[msg.sender] = 0; // too late!
}

// FIXED (Checks-Effects-Interactions pattern)
function withdraw() public {
    uint amount = balances[msg.sender];
    balances[msg.sender] = 0;  // state update FIRST
    (bool success,) = msg.sender.call{value: amount}("");
}
```

**2. Integer Overflow/Underflow**: Fixed in Solidity 0.8+ (auto-reverts)

**3. tx.origin vs msg.sender**:
```solidity
// VULNERABLE: use msg.sender, NOT tx.origin for auth
require(tx.origin == owner); // can be exploited via phishing contract
```

**4. Front-running**: Miners see pending txs and can reorder

**5. Unchecked return values**: Always check `.call()` return value

**6. Selfdestruct**: Can force-send ETH to any contract

---

### Q9. What is the difference between `transfer`, `send`, and `call` for ETH transfers?
**Answer:**
| Method | Gas | Reverts on fail | Recommended |
|---|---|---|---|
| `transfer` | 2300 gas stipend | Yes (auto) | No (deprecated) |
| `send` | 2300 gas stipend | No (returns bool) | No |
| `call` | All gas (configurable) | No (returns bool) | **Yes** |

```solidity
// Recommended pattern
(bool success, ) = recipient.call{value: amount}("");
require(success, "Transfer failed");
```

---

### Q10. What are ERC standards? Explain ERC-20 and ERC-721.
**Answer:**

**ERC-20** (Fungible Token):
```solidity
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}
```

**ERC-721** (NFT — Non-Fungible Token):
- Each token has a unique `tokenId`
- `ownerOf(tokenId)`, `safeTransferFrom`, `tokenURI`
- Used for unique assets: art, real estate, certificates

**ERC-1155**: Multi-token standard — both fungible and non-fungible in one contract

---

### Q11. What are upgradeable smart contracts? How do proxies work?
**Answer:**
Smart contracts are immutable by default. Proxy pattern separates logic from storage:

**Transparent Proxy Pattern** (OpenZeppelin):
- `Proxy` contract: Holds state, delegates calls to `Implementation`
- `Implementation` contract: Contains logic (no state)
- `Admin` can upgrade by pointing proxy to new implementation
- Uses `delegatecall` — runs implementation code in proxy's context

```solidity
// Proxy delegates all calls
fallback() external payable {
    address impl = implementation;
    assembly {
        calldatacopy(0, 0, calldatasize())
        let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
        returndatacopy(0, 0, returndatasize())
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```

**UUPS Proxy**: Upgrade logic lives in implementation (more gas efficient)
**Beacon Proxy**: Multiple proxies share one implementation pointer

---

### Q12. What is `delegatecall` and how is it different from `call`?
**Answer:**
- **`call`**: Executes code in the **callee's** context (uses callee's storage, msg.sender = caller contract)
- **`delegatecall`**: Executes code in the **caller's** context (uses **caller's** storage, msg.sender = original EOA)

Critical for proxy patterns — the proxy's storage is modified using the implementation's code.

---

### Q13. Explain events and logs in Solidity.
**Answer:**
```solidity
event Transfer(address indexed from, address indexed to, uint256 value);

// Emit
emit Transfer(msg.sender, to, amount);
```
- Stored in transaction receipt logs (not contract storage — cheaper)
- `indexed` parameters → can be filtered/searched (up to 3 indexed)
- Frontend listens via `ethers.js` or `web3.js`
- Cannot be read by other contracts (off-chain only)

---

### Q14. What is a modifier in Solidity?
**Answer:**
```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _; // function body executes here
}

function changeOwner(address newOwner) public onlyOwner {
    owner = newOwner;
}
```
Used for access control, reentrancy guards, input validation.

---

### Q15. How do you test smart contracts? (Hardhat/Truffle + Slither)
**Answer:**

**Hardhat** (most common now):
```javascript
const { expect } = require("chai");

describe("Token", function() {
    it("Should transfer tokens", async function() {
        const [owner, addr1] = await ethers.getSigners();
        const Token = await ethers.getContractFactory("Token");
        const token = await Token.deploy(1000);
        
        await token.transfer(addr1.address, 50);
        expect(await token.balanceOf(addr1.address)).to.equal(50);
    });
});
```

**Slither** (static analysis):
```bash
slither . --detect reentrancy-eth,unprotected-upgrade
```
- Detects: reentrancy, unprotected functions, integer overflow, etc.

**Foundry**: Newer, tests written in Solidity, very fast

---

### Q16. What is Polygon and its Layer 2 solutions?
**Answer:**
- **Polygon PoS**: Sidechain with its own validators, bridges to Ethereum
- **zkEVM (ZK Rollups)**: Bundles txs off-chain, generates ZK proof, posts to L1. EVM-compatible
- **Optimistic Rollups (Optimism, Arbitrum)**: Assume txs valid, 7-day challenge period
- **zkSync, StarkNet**: Other ZK rollup implementations

**Polygon SDK (now Polygon Edge)**: Framework for building EVM-compatible chains

PwC may ask about L2 for reducing gas costs in enterprise solutions.

---

### Q17. What is IPFS and how is it used with blockchain?
**Answer:**
- InterPlanetary File System — decentralized file storage
- Files addressed by content hash (CID)
- Pattern: Store large data (images, documents) on IPFS, store **CID hash** on-chain
- Immutable content addressing — if file changes, hash changes
- Used with NFTs: `tokenURI` returns IPFS URL
- Pinning services (Pinata, Infura IPFS) ensure persistence

---

### Q18. How do you interact with a deployed smart contract via Node.js?
**Answer:**
```javascript
const { ethers } = require("ethers");

// Connect to network
const provider = new ethers.JsonRpcProvider("https://rpc-url");
const signer = new ethers.Wallet(privateKey, provider);

// Contract instance
const contract = new ethers.Contract(contractAddress, ABI, signer);

// Read (free)
const balance = await contract.balanceOf(address);

// Write (costs gas)
const tx = await contract.transfer(recipient, amount);
await tx.wait(); // wait for confirmation
```

---

### Q19. What is Hardhat vs Truffle vs Foundry?
**Answer:**
| | Hardhat | Truffle | Foundry |
|---|---|---|---|
| Language | JS/TS | JS | Solidity/Rust |
| Speed | Fast | Slower | Fastest |
| Debugging | Hardhat console.log | Limited | forge test -vvv |
| Network fork | Yes | Yes | Yes |
| Status | Most popular | Declining | Growing fast |

---

### Q20. What are oracles and why are they needed?
**Answer:**
- Smart contracts can't access off-chain data (prices, weather, APIs)
- **Oracle**: Bridge between blockchain and real world
- **Chainlink**: Most popular decentralized oracle network
- **Price feeds**: `AggregatorV3Interface` for USD/ETH prices
- **VRF**: Verifiable Random Function for provable randomness
- Risk: Oracle manipulation → use decentralized oracle networks

---

## ENTERPRISE ETHEREUM / BESU SPECIFIC

### Q21. How does Hyperledger Besu handle privacy?
**Answer:**
- **Tessera**: Private transaction manager (Java)
- Private txs: Payload encrypted, stored in Tessera, only hash on-chain
- `private_sendTransaction` RPC call
- **Privacy Groups**: Define which nodes can see private data
- **On-chain Privacy Groups (Pantheon)**: Managed via smart contract

---

### Q22. What consensus algorithms does Besu support?
**Answer:**
- **QBFT**: Recommended for production private networks. BFT, deterministic finality
- **IBFT 2.0**: BFT, predecessor to QBFT
- **Clique**: PoA (Proof of Authority), simpler but CFT only
- **Ethash**: PoW (for testing Ethereum mainnet behavior)
- Validator sets managed on-chain or via genesis config

---

### Q23. How do you configure a private Besu network?
**Answer:**
1. Genesis file with `config.chainId`, consensus engine config, initial validators
2. Node key generation (`besu --generate-blockchain-config`)
3. Static node configuration or bootnode setup
4. P2P discovery configuration
5. JSON-RPC API enablement
6. Tessera setup for privacy (optional)

---

### Q24. What is EIP-712 (Structured Data Signing)?
**Answer:**
- Standard for signing structured data (not just raw bytes)
- Enables human-readable signing in wallets (MetaMask shows structured data)
- Used in: Permit (gasless ERC-20 approvals), Seaport (NFT marketplace), meta-transactions
- Domain separator prevents replay attacks across chains/contracts

---

### Q25. What is a flash loan attack? How do you prevent it?
**Answer:**
Flash loan: Borrow unlimited funds within one transaction, return by end of tx.

**Attack pattern**: Borrow → Manipulate price oracle → Exploit → Repay

**Prevention**:
- Use time-weighted average prices (TWAP) instead of spot prices
- Use Chainlink price feeds
- Add slippage checks
- Reentrancy guards
- Check-effects-interactions pattern
