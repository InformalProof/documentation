# Lendi x ReinieraOS Technology Stack

> Every tool, library, protocol, and service used, with rationale and integration details.

---

## Overview Map

```
CRYPTOGRAPHY          BLOCKCHAIN            PAYMENTS              UX
─────────────         ──────────────        ─────────────         ──────────────
Fhenix CoFHE          Arbitrum Sepolia      Privara               ZeroDev AA
@cofhe/sdk            Arbitrum One          ReinieraOS            platform-modules
cofhe-hardhat-plugin  Circle CCTP V2        @reineira-os/sdk      WebLLM
FHE.sol library       viem                  USDC / BRLA / MXNe    React + hooks
```

---

## 1. Fhenix CoFHE — FHE Computation Engine

**What it is:** An off-chain FHE coprocessor that executes homomorphic operations on encrypted data without exposing plaintext at any step.

**Why we use it:** FHE is the only technology that allows computing on encrypted data while keeping it encrypted. CoFHE makes this practical for EVM smart contracts by offloading computation off-chain while maintaining on-chain verifiability.

**How it works:**
1. Smart contract emits FHE operation request event
2. Slim Listener detects event, routes to FheOS engine
3. FheOS executes homomorphic operation on ciphertext off-chain
4. Result Processor publishes encrypted result back on-chain
5. Threshold Network handles decryption via MPC when authorized

**Key specs:**
- Live on: Sepolia, Arbitrum Sepolia, Base Sepolia
- Decryption: 50x faster than competing approaches
- Gas model: FHE ops charged on host chain at medium gas tier
- Security: Threshold decryption via MPC — key never fully reconstructed

**Components used in Lendi:**

| Component | Role |
|---|---|
| TaskManager | On-chain entry point for FHE operation requests |
| Slim Listener | Off-chain event monitor — routes ops to FheOS |
| FheOS | Computation engine — executes ops on ciphertexts |
| Result Processor | Publishes encrypted results back on-chain |
| Threshold Network | MPC-based decryption — key split across nodes |

**Docs:** https://cofhe-docs.fhenix.zone

---

## 2. @fhenixprotocol/cofhe-contracts — Solidity FHE Library

**What it is:** Solidity library exposing encrypted types and FHE operations for smart contracts.

**Why we use it:** Provides the core primitives (`euint64`, `ebool`, `eaddress`, `InEuint64`) and operations (`FHE.add`, `FHE.gte`, `FHE.select`, `FHE.allow`) that Lendi is built on.

**Installation:**
```bash
npm install @fhenixprotocol/cofhe-contracts
```

**Import:**
```solidity
import {FHE, euint64, InEuint64, ebool} from "@fhenixprotocol/cofhe-contracts/FHE.sol";
```

**Encrypted types used:**

| Type | Description | Used for |
|---|---|---|
| `euint64` | Encrypted 64-bit unsigned integer | Monthly income, threshold |
| `ebool` | Encrypted boolean | Qualification result |
| `eaddress` | Encrypted address | Borrower/lender in escrow |
| `InEuint64` | Encrypted input from client | Function parameter type |

**Operations used:**

| Operation | Description | Gas |
|---|---|---|
| `FHE.asEuint64(InEuint64)` | Convert client input to stored type | Medium |
| `FHE.asEuint64(uint64)` | Trivial encrypt a constant | Low |
| `FHE.add(a, b)` | Add two encrypted values | Medium |
| `FHE.gte(a, b)` | Compare two encrypted values | Medium |
| `FHE.select(cond, a, b)` | Conditional select without branching | Medium |
| `FHE.allowThis(h)` | Grant contract access to handle | Minimal |
| `FHE.allow(h, addr)` | Grant specific address decrypt permission | Minimal |
| `FHE.getDecryptResultSafe(h)` | Read decrypted result with validity check | Minimal |

**GitHub:** https://github.com/FhenixProtocol/cofhe-contracts

---

## 3. @cofhe/sdk — Client-Side TypeScript SDK

**What it is:** TypeScript SDK for client-side encryption, permit management, and decryption.

**Why we use it:** All encryption happens client-side — plaintext never leaves the user's device before being encrypted. The SDK handles the FHE key management, ZK proof generation for inputs, and permit-based decryption.

> ⚠️ Use `@cofhe/sdk` — not the deprecated `cofhejs`.

**Installation:**
```bash
npm install @cofhe/sdk
```

**Key functions used in Lendi:**

```typescript
import {
  createCofheConfig,
  createCofheClient,
  Encryptable,
  FheTypes
} from '@cofhe/sdk/web' // browser
// or '@cofhe/sdk/node' for Node.js

// Initialize
const config = createCofheConfig({ supportedChains: [chains.arbitrumSepolia] })
const client = await createCofheClient(config)
await client.connect(publicClient, walletClient)

// Encrypt income before sending to contract
const [encAmount] = await client
  .encryptInputs([Encryptable.uint64(incomeAmount)])
  .execute()

// Worker views own income in UI (device RAM only)
const { decryptedValue } = await client
  .decryptForView(incomeHandle, FheTypes.Uint64)
  .execute()

// Generate verifiable proof result for lender
const { ctHash, decryptedValue, signature } = await client
  .decryptForTx(qualifiesHandle)
  .withoutPermit()
  .execute()

// Permit management
const permit = await client.permits.getOrCreateSelfPermit()
```

**Docs:** https://cofhe-docs.fhenix.zone/client-sdk/introduction/overview.md

---

## 4. @cofhe/hardhat-plugin — Local Development Plugin

**What it is:** Hardhat plugin that injects mock CoFHE contracts for local development and testing.

**Why we use it:** Allows testing FHE contracts locally without testnet access. Mock contracts execute FHE operations symbolically on plaintext, enabling fast iteration.

**Installation:**
```bash
npm install @cofhe/hardhat-plugin
```

**Key testing utilities:**

```typescript
import { hre } from 'hardhat'

// Read plaintext value of encrypted handle (tests only)
const plainIncome = await hre.cofhe.mocks.getPlaintext(incomeHandle)

// Assert encrypted value equals expected
await hre.cofhe.mocks.expectPlaintext(incomeHandle, 1200_000000n)

// Create test client with batteries included
const client = await hre.cofhe.createClientWithBatteries(signer)
```

**Docs:** https://cofhe-docs.fhenix.zone/client-sdk/hardhat-plugin/getting-started.md

---

## 5. Privara + ReinieraOS — Payment Rails

**What it is:** Privacy-preserving payment infrastructure and conditional settlement engine built for the Fhenix ecosystem.

**Why we use it:** Provides the verified on-chain income events that Lendi reads and encrypts. Also provides the escrow infrastructure, protection pools, and ZeroDev integration that turns Lendi into a full lending product.

### ReinieraOS SDK

**Installation:**
```bash
npm install @reineira-os/sdk
```

**Usage in Lendi:**

```typescript
import { ReinieraClient } from '@reineira-os/sdk'

const client = new ReinieraClient({ network: 'arbitrum-sepolia' })

// Listen for incoming payments (primary income source)
client.onPaymentReceived(workerAddress, async (payment) => {
  const [encAmount] = await cofheClient
    .encryptInputs([Encryptable.uint64(payment.amount)])
    .execute()
  await informalProofContract.write.recordIncome([encAmount])
})
```

### ReinieraOS Components Used

| Component | Role in Lendi |
|---|---|
| `ConfidentialEscrow` | Loan lifecycle — holds and releases funds |
| `IConditionResolver` | Interface that `LendiGate` implements |
| `ProtectionPool` | Lender coverage against defaults |
| `judge()` | Automated encrypted default resolution |
| `platform-modules` | iOS/Web UI (ships Wave 2) |
| `reineira-atlas` | AI agents for strategy and compliance (live now) |
| `reineira-code` | AI-assisted Solidity contract generation |

**Gate Plugin Pattern:**
```typescript
// LendiGate implements IConditionResolver
// ReinieraOS calls it before releasing escrow funds
interface IConditionResolver {
    function isConditionMet(bytes32 escrowId) external returns (bool);
}
```

**Docs:** https://docs.reineira.xyz/docs/build
**SDK:** https://www.npmjs.com/package/@reineira-os/sdk
**MCP Server:** https://docs.reineira.xyz/docs/reference/mcp-server

---

## 6. Circle CCTP V2 — Cross-Chain Settlement

**What it is:** Circle's Cross-Chain Transfer Protocol for native USDC movement across EVM chains.

**Why we use it:** Enables borrowers and lenders on different chains to participate in the same loan. Loan funds can be sourced on Ethereum and delivered on Arbitrum, or any CCTP-supported combination.

**Resilience model:**
- 3-tier operator fallback for relay
- 0–60s: assigned operator exclusive relay
- 60–600s: any staked operator can relay
- 600s+: anyone can relay — no stake required
- No loan ever permanently blocked

---

## 7. ZeroDev — Account Abstraction

**What it is:** ERC-4337 account abstraction infrastructure with smart accounts, session keys, and social login.

**Why we use it:** Informal workers should not need to understand wallets, gas, or seed phrases. ZeroDev creates a smart account automatically on social login, handles gas sponsorship, and enables batched transactions.

**What it enables for Lendi:**
- Social login (Google, email) → smart account created automatically
- No ETH needed for gas — sponsored by the protocol
- Session keys — workers approve income recording once, not every transaction
- Batched transactions — record multiple income events in one tx

**Integration:** Provided via `platform-modules` from ReinieraOS (ships Wave 2).

---

## 8. viem — Ethereum Client Library

**What it is:** TypeScript library for Ethereum interactions.

**Why we use it:** Read USDC Transfer events directly from Arbitrum Sepolia as the fallback income source. No API keys, no intermediaries — pure on-chain data.

```typescript
import { createPublicClient, parseAbiItem, http } from 'viem'
import { arbitrumSepolia } from 'viem/chains'

const client = createPublicClient({
  chain:     arbitrumSepolia,
  transport: http()
})

const logs = await client.getLogs({
  address: USDC_ADDRESS,
  event:   parseAbiItem(
    'event Transfer(address indexed from, address indexed to, uint256 value)'
  ),
  args:      { to: workerAddress },
  fromBlock: startOfMonthBlock
})
```

---

## 9. WebLLM — Browser-Local AI

**What it is:** Open-source library for running LLM inference entirely in the browser using WebGPU.

**Why we use it:** The AI financial advisor needs to see the worker's decrypted income to give useful advice. But sending that data to any server — OpenAI, Anthropic, our own — would break the privacy guarantee. WebLLM runs the model locally: data never leaves the device.

**Model used:** `Llama-3.2-3B-Instruct-q4f32_1-MLC`
- Size: ~2GB (one-time download, cached by browser)
- Performance: runs on consumer hardware via WebGPU
- Latency: 2–8 seconds per response

```typescript
import { CreateMLCEngine } from '@mlc-ai/web-llm'

const engine = await CreateMLCEngine('Llama-3.2-3B-Instruct-q4f32_1-MLC', {
  initProgressCallback: (progress) => console.log(progress)
})

const response = await engine.chat.completions.create({
  messages: [{
    role:    'system',
    content: 'You are a concise financial advisor for informal workers in LATAM.'
  }, {
    role:    'user',
    content: `My verified monthly income is $${income} USDC. 
              I want to apply for a $300 loan at 8% over 3 months. 
              Is this advisable?`
  }],
  max_tokens:  300,
  temperature: 0.7
})
```

**Privacy guarantee:** Decrypted income exists only in browser RAM during the session. No network call is made. Session data cleared on tab close.

---

## 10. React + @cofhe/react — Frontend

**What it is:** React hooks library for CoFHE interactions in frontend applications.

**Why we use it:** Wraps the CoFHE SDK in React-friendly hooks, handling wallet connection, encryption state, and permit management automatically.

**Key hooks:**

```typescript
import {
  useEncrypt,
  useDecrypt,
  useWrite,
  useCofheClient
} from '@cofhe/react'

// Worker dashboard
const { encrypt }   = useEncrypt()
const { decrypt }   = useDecrypt()
const { write }     = useWrite(contract, 'recordIncome')

// Lender view
const { write: prove } = useWrite(contract, 'proveIncome')
```

---

## 11. Hardhat — Development Environment

**What it is:** Ethereum development environment for compiling, testing, and deploying contracts.

**Why we use it:** Standard toolchain for EVM development. CoFHE provides a first-class Hardhat plugin.

**Key configuration:**
```typescript
// hardhat.config.ts
import '@cofhe/hardhat-plugin'

const config: HardhatUserConfig = {
  solidity: {
    version: '0.8.24',
    settings: {
      evmVersion: 'cancun', // REQUIRED for CoFHE
      optimizer: { enabled: true, runs: 200 }
    }
  }
}
```

---

## 12. ERC Standards Used

| Standard | Purpose |
|---|---|
| **ERC-7201** | Namespaced storage — prevents storage collisions on contract upgrades |
| **ERC-2771** | Meta-transactions — workers send gasless transactions via trusted forwarder |
| **ERC-4337** | Account abstraction — ZeroDev smart accounts (via platform-modules) |
| **EIP-712** | Typed structured data signing — off-chain loan term confirmation |

---

## 13. Supported Stablecoins

| Token | Network | Chain ID | Purpose |
|---|---|---|---|
| USDC | Arbitrum Sepolia | 421614 | Primary income + loan currency |
| USDT | Arbitrum Sepolia | 421614 | Alternative income currency |
| BRLA | Polygon | 137 | Brazilian Real-pegged workers |
| MXNe | Base | 8453 | Mexican Peso-pegged workers |
| USDC | Arbitrum One | 42161 | Mainnet target (Wave 4) |

---

## 14. AI Tools for Development

| Tool | Purpose |
|---|---|
| **Neo FHE AI Assistant** | Fhenix-trained AI for FHE contract development. Knows anti-patterns. Use before writing any FHE Solidity. |
| **reineira-atlas** | ReineiraOS AI agents for strategy, compliance, tokenomics, investor readiness. Available now. |
| **reineira-code** | AI-assisted Solidity plugin development — describe what you want, generates contract + tests. |

**Neo FHE:** https://fhenix.notion.site/neo-fhe-ai-assistant
**reineira-atlas:** Available via ReinieraOS partnership
**reineira-code:** https://github.com/ReineiraOS/reineira-code

---

## Dependency Graph

```
Lendi.sol
├── @fhenixprotocol/cofhe-contracts (FHE.sol)
│   └── Fhenix CoFHE coprocessor (off-chain)
│       └── Threshold Network (MPC decryption)
└── IConditionResolver (ReinieraOS interface)

LendiGate.sol
├── Lendi.sol
└── @reineira-os/contracts (IConditionResolver)
    └── ConfidentialEscrow.sol (ReinieraOS)
        ├── ProtectionPool.sol (ReinieraOS)
        └── Circle CCTP V2 (cross-chain)

Frontend
├── @cofhe/sdk/web
├── @cofhe/react
├── @reineira-os/sdk
├── viem
└── @mlc-ai/web-llm (WebLLM)

Testing
├── @cofhe/hardhat-plugin
└── hardhat
```

---

*Lendi | Built on Fhenix CoFHE | Powered by Privara + ReinieraOS*
