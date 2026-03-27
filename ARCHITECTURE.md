# Lendi Technical Architecture

---

## System Overview

```mermaid
graph TB
    subgraph APP["Application Layer"]
        UI[React Frontend]
        AI[WebLLM - Local AI]
        ZD[ZeroDev Smart Account]
    end

    subgraph PROTOCOL["Protocol Layer"]
        IP[Lendi.sol]
        GATE[LendiGate.sol]
        ESCROW[ConfidentialEscrow - ReinieraOS]
        POOL[ProtectionPool - ReinieraOS]
    end

    subgraph INFRA["Infrastructure Layer"]
        COFHE[Fhenix CoFHE]
        PRIVARA[Privara / ReinieraOS]
        CCTP[Circle CCTP v2]
        CHAIN[Arbitrum Sepolia]
    end

    UI --> IP
    UI --> ESCROW
    AI --> UI
    ZD --> UI

    IP --> COFHE
    GATE --> IP
    ESCROW --> GATE
    ESCROW --> POOL
    ESCROW --> CCTP

    COFHE --> CHAIN
    PRIVARA --> IP
    CCTP --> CHAIN
```

---

## Income Recording Flow

```mermaid
sequenceDiagram
    participant W as Worker Device
    participant P as Privara/ReinieraOS
    participant SDK as @cofhe/sdk
    participant C as Lendi.sol
    participant FHE as CoFHE Engine

    P->>W: Payment received event (plaintext on-chain)
    W->>SDK: encryptInputs([amount])
    Note over SDK: Encrypted in browser — plaintext gone
    SDK->>W: InEuint64
    W->>C: recordIncome(InEuint64)
    C->>FHE: FHE.asEuint64 + FHE.add
    Note over FHE: Addition on ciphertexts — no decryption
    FHE->>C: euint64 handle updated
    C->>C: FHE.allowThis + FHE.allow(worker)
```

---

## Credit Verification Flow

```mermaid
sequenceDiagram
    participant L as Lender
    participant GATE as LendiGate.sol
    participant IP as Lendi.sol
    participant FHE as CoFHE Engine
    participant ESC as ConfidentialEscrow

    L->>ESC: Create escrow + fund $300 USDC
    ESC->>GATE: isConditionMet(escrowId)?
    GATE->>IP: proveIncome(worker, threshold)
    IP->>FHE: FHE.gte(monthlyIncome, required)
    Note over FHE: Two ciphertexts compared — neither decrypted
    FHE->>IP: ebool handle
    IP->>IP: FHE.allow(ebool, lender + worker)
    GATE->>FHE: getDecryptResultSafe
    FHE->>GATE: true / false
    GATE->>ESC: condition met
    ESC->>ESC: FHE.select(true, paidAmount, zero)
    ESC->>L: Funds released to worker
    Note over L: Lender never saw the income amount
```

---

## AI Advisor Flow

```mermaid
sequenceDiagram
    participant W as Worker Device RAM
    participant SDK as @cofhe/sdk
    participant TN as Threshold Network
    participant LLM as WebLLM local

    W->>SDK: decryptForView(incomeHandle)
    SDK->>TN: Decrypt request + permit
    Note over TN: MPC — key never fully reconstructed
    TN->>W: decryptedValue in browser RAM only
    W->>LLM: Query with local data
    Note over LLM: Runs on device GPU via WebGPU
    Note over LLM: Zero network calls
    LLM->>W: Financial advice
    Note over W: Session ends — RAM cleared — zero leakage
```

---

## ACL Model

```mermaid
graph LR
    INC[monthlyIncome euint64]
    QUAL[qualifies ebool]

    INC -- FHE.allowThis --> CONTRACT[Contract]
    INC -- FHE.allow --> WORKER[Worker]
    QUAL -- FHE.allow --> LENDER[Lender]
    QUAL -- FHE.allow --> WORKER
```

Nobody else can decrypt anything. Not the protocol. Not the public.

---

## Core Contracts

### Lendi.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint64, InEuint64, ebool} from "@fhenixprotocol/cofhe-contracts/FHE.sol";

contract Lendi {
    mapping(address => euint64) private monthlyIncome;
    mapping(address => bool)    public  registeredWorkers;
    mapping(address => bool)    public  registeredLenders;
    mapping(bytes32 => address) public  escrowToWorker;
    mapping(bytes32 => uint64)  public  escrowToThreshold;

    function recordIncome(InEuint64 calldata encAmount) external onlyWorker {
        euint64 amount = FHE.asEuint64(encAmount);
        monthlyIncome[msg.sender] = FHE.add(monthlyIncome[msg.sender], amount);
        FHE.allowThis(monthlyIncome[msg.sender]); // MANDATORY — #1 bug if forgotten
        FHE.allow(monthlyIncome[msg.sender], msg.sender);
    }

    function proveIncome(
        address worker,
        uint64  threshold
    ) external onlyLender returns (ebool) {
        euint64 required  = FHE.asEuint64(threshold);
        ebool   qualifies = FHE.gte(monthlyIncome[worker], required);
        FHE.allow(qualifies, msg.sender); // lender sees result
        FHE.allow(qualifies, worker);     // worker sees result
        return qualifies;
    }
}
```

### LendiGate.sol

```solidity
// Plugs into ReinieraOS as IConditionResolver
contract LendiGate is IConditionResolver {
    function isConditionMet(bytes32 escrowId) external returns (bool) {
        address worker    = informalProof.escrowToWorker(escrowId);
        uint64  threshold = informalProof.escrowToThreshold(escrowId);
        ebool   qualifies = informalProof.proveIncome(worker, threshold);
        (uint256 result, bool valid) = FHE.getDecryptResultSafe(qualifies);
        require(valid, "Not ready");
        return result == 1;
    }
}
```

---

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Smart contracts | Solidity 0.8.24 | Core FHE logic |
| FHE library | `@fhenixprotocol/cofhe-contracts` | `euint64`, `ebool`, operations |
| Client SDK | `@cofhe/sdk` — not deprecated cofhejs | Encrypt, decrypt, permits |
| Payment rails | `@reineira-os/sdk` | Income capture, escrow, settlement |
| On-chain reads | `viem` | USDC Transfer event fallback |
| AI advisor | `@mlc-ai/web-llm` | Local LLM, zero data leakage |
| Frontend | React + `@cofhe/react` | Worker + lender UI |
| Accounts | ZeroDev (Wave 2) | Social login, sponsored gas |
| Testing | Hardhat + `@cofhe/hardhat-plugin` | Mock FHE locally |
| Network | Arbitrum Sepolia -> public mainnet -> Fhenix mainnet (autumn 2026) | Phased deployment |

---

## Critical Rules

> ⚠️ `FHE.allowThis()` is mandatory after every FHE mutation. Forgetting it is the #1 bug.

> ⚠️ No `if/else` on encrypted values. Use `FHE.select()` instead.

> ⚠️ `evmVersion: 'cancun'` required in `hardhat.config.ts`.

> ⚠️ Use `@cofhe/sdk`, not deprecated `cofhejs`.

---

*Lendi | Fhenix CoFHE | Privara + ReinieraOS*
