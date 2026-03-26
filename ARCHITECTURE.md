# ARCHITECTURE.md
## InformalProof × ReinieraOS — Technical Architecture

> Full-stack architecture: components, contracts, FHE operations, ACL model, and integration points.

---

## System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 3 — APPLICATION                                               │
│  ZeroDev smart accounts (no wallet / no gas for end user)            │
│  platform-modules iOS + Web UI (ReineiraOS, ships Wave 2)            │
│  WebLLM — AI financial advisor running entirely in browser           │
│  InformalProof React frontend (Wave 1 demo dApp)                     │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │ @cofhe/sdk · @reineira-os/sdk · viem
                                 ↓
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 2 — PROTOCOL                                                  │
│  InformalProof.sol      — income recording + ACL                     │
│  InformalProofGate.sol  — IConditionResolver for ReinieraOS          │
│  ConfidentialEscrow.sol — loan lifecycle (ReinieraOS)                │
│  ProtectionPool.sol     — lender coverage + judge() (ReinieraOS)     │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │ CoFHE coprocessor events
                                 ↓
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — INFRASTRUCTURE                                            │
│  Fhenix CoFHE     — FHE computation engine (off-chain compute)       │
│  Circle CCTP V2   — cross-chain USDC settlement                      │
│  ERC-7201         — namespaced encrypted storage                     │
│  ERC-2771         — meta-transactions (gasless UX)                   │
│  Arbitrum Sepolia — host chain (Waves 1–3)                           │
│  Arbitrum Mainnet — target (Wave 4+)                                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Core Contracts

### 1. InformalProof.sol

Primary FHE contract. Handles encrypted income accumulation and threshold verification.

**Responsibilities:**
- Accept encrypted income inputs from workers
- Accumulate income history as `euint64` ciphertext
- Respond to verification requests with `ebool` only — never the income amount
- Enforce ACL — three roles: worker, lender, protocol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint64, InEuint64, ebool} from "@fhenixprotocol/cofhe-contracts/FHE.sol";

contract InformalProof {

    // ── Encrypted State ───────────────────────────────────────────────
    mapping(address => euint64) private monthlyIncome;
    mapping(address => euint64) private txCount;
    mapping(address => uint256) public  lastResetTimestamp;

    // ── Plaintext Registry ────────────────────────────────────────────
    mapping(address => bool)    public registeredWorkers;
    mapping(address => bool)    public registeredLenders;
    mapping(bytes32 => address) public escrowToWorker;
    mapping(bytes32 => uint64)  public escrowToThreshold;

    address public owner;
    uint256 public constant RESET_PERIOD = 30 days;

    // ── Events (no amounts ever emitted) ─────────────────────────────
    event WorkerRegistered(address indexed worker);
    event LenderRegistered(address indexed lender);
    event IncomeRecorded(address indexed worker, uint256 timestamp);
    event ProofRequested(address indexed lender, address indexed worker);
    event MonthlyReset(address indexed worker, uint256 timestamp);
    event EscrowLinked(bytes32 indexed escrowId, address indexed worker);

    // ── Modifiers ─────────────────────────────────────────────────────
    modifier onlyOwner()  { require(msg.sender == owner, "Not owner"); _; }
    modifier onlyWorker() { require(registeredWorkers[msg.sender], "Not worker"); _; }
    modifier onlyLender() { require(registeredLenders[msg.sender], "Not lender"); _; }

    // ── Registration ──────────────────────────────────────────────────

    function registerWorker() external {
        registeredWorkers[msg.sender] = true;
        emit WorkerRegistered(msg.sender);
    }

    function registerLender(address lender) external onlyOwner {
        registeredLenders[lender] = true;
        emit LenderRegistered(lender);
    }

    // ── Core: Income Recording ────────────────────────────────────────

    function recordIncome(InEuint64 calldata encAmount) external onlyWorker {
        euint64 amount = FHE.asEuint64(encAmount);

        monthlyIncome[msg.sender] = FHE.add(monthlyIncome[msg.sender], amount);
        FHE.allowThis(monthlyIncome[msg.sender]);
        FHE.allow(monthlyIncome[msg.sender], msg.sender);

        txCount[msg.sender] = FHE.add(txCount[msg.sender], FHE.asEuint64(1));
        FHE.allowThis(txCount[msg.sender]);

        emit IncomeRecorded(msg.sender, block.timestamp);
    }

    // ── Core: Credit Verification ─────────────────────────────────────

    function proveIncome(
        address worker,
        uint64  threshold
    ) external onlyLender returns (ebool) {
        require(registeredWorkers[worker], "Worker not registered");

        euint64 required  = FHE.asEuint64(threshold);
        ebool   qualifies = FHE.gte(monthlyIncome[worker], required);

        FHE.allow(qualifies, msg.sender); // lender sees result
        FHE.allow(qualifies, worker);     // worker sees result

        emit ProofRequested(msg.sender, worker);
        return qualifies;
    }

    // ── Gate Integration ──────────────────────────────────────────────

    function linkEscrow(
        bytes32 escrowId,
        address worker,
        uint64  threshold
    ) external onlyLender {
        escrowToWorker[escrowId]    = worker;
        escrowToThreshold[escrowId] = threshold;
        emit EscrowLinked(escrowId, worker);
    }

    // ── Monthly Reset ─────────────────────────────────────────────────

    function resetMonthlyIncome() external onlyWorker {
        require(
            block.timestamp >= lastResetTimestamp[msg.sender] + RESET_PERIOD,
            "Reset period not elapsed"
        );
        monthlyIncome[msg.sender] = FHE.asEuint64(0);
        FHE.allowThis(monthlyIncome[msg.sender]);
        FHE.allow(monthlyIncome[msg.sender], msg.sender);
        lastResetTimestamp[msg.sender] = block.timestamp;
        emit MonthlyReset(msg.sender, block.timestamp);
    }
}
```

---

### 2. InformalProofGate.sol

Condition resolver that connects InformalProof to ReinieraOS escrows. Implements `IConditionResolver` so ReinieraOS calls it as a gate before releasing funds.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, ebool} from "@fhenixprotocol/cofhe-contracts/FHE.sol";
import {IConditionResolver} from "@reineira-os/contracts/interfaces/IConditionResolver.sol";
import {InformalProof} from "./InformalProof.sol";

contract InformalProofGate is IConditionResolver {

    InformalProof public immutable informalProof;

    constructor(address _informalProof) {
        informalProof = InformalProof(_informalProof);
    }

    /// @notice Called by ReinieraOS ConfidentialEscrow before releasing funds
    function isConditionMet(bytes32 escrowId) external override returns (bool) {
        address worker    = informalProof.escrowToWorker(escrowId);
        uint64  threshold = informalProof.escrowToThreshold(escrowId);

        require(worker != address(0), "Escrow not linked");

        // FHE comparison — income and threshold never revealed
        ebool qualifies = informalProof.proveIncome(worker, threshold);

        // Decrypt for gate evaluation only
        (uint256 result, bool valid) = FHE.getDecryptResultSafe(qualifies);
        require(valid, "Decrypt result not ready");
        return result == 1;
    }
}
```

---

### 3. ConfidentialEscrow.sol (ReinieraOS)

Manages full loan lifecycle. InformalProofGate plugs in as condition resolver.

```solidity
// Key encrypted struct — everything sensitive is ciphertext
struct Escrow {
    eaddress owner;       // borrower wallet — encrypted
    eaddress caller;      // lender wallet   — encrypted
    euint64  amount;      // loan amount     — encrypted
    euint64  paidAmount;  // funded amount   — encrypted
    ebool    isRedeemed;  // status          — encrypted
    bool     exists;      // only public field
}

// Silent failure — never reverts, never leaks why it failed
function redeem(bytes32 escrowId) external {
    bool conditionMet = gate.isConditionMet(escrowId);

    euint64 transferAmount = FHE.select(
        FHE.asEbool(conditionMet),
        escrows[escrowId].paidAmount,
        FHE.asEuint64(0)          // transfers 0 instead of reverting
    );
    _transfer(escrows[escrowId].owner, transferAmount);
}
```

---

### 4. ProtectionPool.sol (ReinieraOS)

Lender coverage via FHE-encrypted risk models.

- `FHE.select` for premium calculation on encrypted risk score
- `judge()` — encrypted verdict on default claims, no manual process
- Pool isolation — one pool's exhaustion cannot affect others
- Encrypted pool balance — prevents bank runs
- Yield for stakers from premiums collected

---

## ACL Model — Three-Role System

```
HANDLE: monthlyIncome[worker]  (euint64)
├── FHE.allowThis(h)     → Contract reuses handle in future tx
├── FHE.allow(h, worker) → Worker decrypts via decryptForView
└── DENIED TO: lender · protocol · public

HANDLE: qualifies  (ebool from proveIncome)
├── FHE.allow(h, lender) → Lender decrypts boolean result
├── FHE.allow(h, worker) → Worker sees own qualification
└── DENIED TO: other lenders · public

HANDLE: Escrow.amount  (euint64 in ReinieraOS)
├── FHE.allow(h, borrower) → Borrower sees loan amount
├── FHE.allow(h, lender)   → Lender sees loan amount
└── DENIED TO: third parties · protection pool
```

### ACL Function Reference

| Function | Grants | When used in InformalProof |
|---|---|---|
| `FHE.allowThis(h)` | Contract reuses handle in future tx | After every `FHE.add()` — MANDATORY |
| `FHE.allow(h, addr)` | Specific address can decrypt | Worker on income; lender+worker on ebool |
| `FHE.allowSender(h)` | Current tx sender can decrypt | Immediate feedback flows |
| `FHE.allowGlobal(h)` | Anyone can decrypt | NOT USED — breaks privacy |

> ⚠️ Forgetting `FHE.allowThis()` after any FHE operation is the #1 integration bug. The handle becomes inaccessible on the next transaction.

> ⚠️ No `if/else` on encrypted values. Use `FHE.select()` for all conditional logic on encrypted types.

---

## FHE Operations

| Operation | Used for | Gas tier |
|---|---|---|
| `FHE.asEuint64(InEuint64)` | Convert user input to stored type | Medium |
| `FHE.asEuint64(uint64)` | Trivial encryption of threshold constant | Low |
| `FHE.add(euint64, euint64)` | Accumulate income | Medium |
| `FHE.gte(euint64, euint64)` | Compare income vs threshold | Medium |
| `FHE.select(ebool, euint64, euint64)` | Silent failure pattern | Medium |

---

## CoFHE Operation Lifecycle

```
Contract calls FHE.add() / FHE.gte()
        ↓
TaskManager emits FHE operation event on-chain
        ↓
Slim Listener (off-chain) detects event
        ↓
Routes to FheOS computation engine
        ↓
FheOS executes homomorphic operation on ciphertext
        ↓
Result Processor publishes encrypted result on-chain
        ↓
If decryption requested:
  Threshold Network processes via MPC
  Key split across nodes — never fully reconstructed
        ↓
Decrypted result → ONLY to address with FHE.allow()
```

---

## Income Sources

### Source A — Privara / ReinieraOS (Primary)

```typescript
import { createCofheClient, Encryptable } from '@cofhe/sdk/web'
import { ReinieraClient } from '@reineira-os/sdk'

const reineira = new ReinieraClient({ network: 'arbitrum-sepolia' })

reineira.onPaymentReceived(workerAddress, async (payment) => {
  const [encAmount] = await cofhe
    .encryptInputs([Encryptable.uint64(payment.amount)])
    .execute()
  await contract.write.recordIncome([encAmount])
})
```

### Source B — Direct USDC on-chain (Fallback)

```typescript
import { createPublicClient, parseAbiItem } from 'viem'

const transfers = await client.getLogs({
  address: USDC_ADDRESS,
  event: parseAbiItem(
    'event Transfer(address indexed from, address indexed to, uint256 value)'
  ),
  args:      { to: workerAddress },
  fromBlock: startOfMonthBlock
})
const totalIncome = transfers.reduce((sum, t) => sum + t.args.value!, 0n)
```

### Supported Tokens

| Token | Network | Use case |
|---|---|---|
| USDC | Arbitrum Sepolia | Primary |
| USDT | Arbitrum Sepolia | Alternative |
| BRLA | Polygon | Brazil workers |
| MXNe | Base | Mexico workers |

---

## AI Advisor — Local Privacy Model

```typescript
import { CreateMLCEngine } from '@mlc-ai/web-llm'

// 1. Decrypt in device RAM only
const { decryptedValue } = await cofheClient
  .decryptForView(incomeHandle, FheTypes.Uint64)
  .execute()

// 2. Load local model
const engine = await CreateMLCEngine('Llama-3.2-3B-Instruct-q4f32_1-MLC')

// 3. Query with local data — no server call
const response = await engine.chat.completions.create({
  messages: [{
    role: 'user',
    content: `My monthly income is $${Number(decryptedValue)/1e6} USDC. 
              Should I apply for a $300 loan?`
  }]
})
// Session ends → RAM cleared → zero leakage
```

---

## Network Configuration

```typescript
// hardhat.config.ts
const config: HardhatUserConfig = {
  solidity: {
    version: '0.8.24',
    settings: { evmVersion: 'cancun' } // REQUIRED for CoFHE
  },
  networks: {
    arbitrumSepolia: {
      url:      'https://sepolia-rollup.arbitrum.io/rpc',
      chainId:  421614,
      accounts: [process.env.PRIVATE_KEY!]
    }
  }
}
```

---

## Repository Structure

```
informalproof/
├── contracts/
│   ├── InformalProof.sol
│   ├── InformalProofGate.sol
│   └── interfaces/IInformalProof.sol
├── test/
│   ├── InformalProof.test.ts
│   ├── InformalProofGate.test.ts
│   └── integration/FullFlow.test.ts
├── scripts/
│   ├── deploy.ts
│   └── simulateLoan.ts
├── sdk/                        # Wave 3 — @informalproof/sdk
├── frontend/
│   ├── components/
│   │   ├── WorkerDashboard.tsx
│   │   ├── LoanApplication.tsx
│   │   └── LenderMarketplace.tsx
│   └── hooks/
│       ├── useInformalProof.ts
│       ├── useDecryptedIncome.ts
│       └── useAIAdvisor.ts
├── docs/
│   ├── CONTEXT.md
│   ├── ARCHITECTURE.md
│   ├── TECHNOLOGIES.md
│   ├── PLANIFICATIONWAVES.md
│   └── USERandDATAFLOW.md
└── hardhat.config.ts
```

---

## Security Model

### What FHE Guarantees
- Income amounts never in plaintext on-chain
- Lender provably cannot access income — only boolean result
- Protocol has no server — zero plaintext processed centrally
- Protection pool balance encrypted — prevents bank runs

### Known Limitations (v1)
- `txCount` is not encrypted — observer can count transactions
- Timing of income recording visible on-chain
- Wallet linkage — if wallet is KYC'd elsewhere, identity may be known

### Planned Mitigations (Wave 3+)
- Encrypt `txCount` as `euint32`
- Optional ERC-2771 relayer for wallet anonymity
- Batch income recording to reduce timing leakage

### Trust Assumptions

| Component | If compromised |
|---|---|
| CoFHE / Fhenix | All encrypted values readable |
| InformalProofGate | Wrong gate logic blocks or wrongly approves loans |
| ReinieraOS Escrow | Malicious resolver can lock funds — cannot redirect them |
| Circle CCTP | Invalid USDC mints on destination chain |

---

*InformalProof | Fhenix CoFHE | Privara + ReinieraOS*
