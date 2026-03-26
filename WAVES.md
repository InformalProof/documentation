# PLANIFICATIONWAVES.md
## InformalProof × ReinieraOS — Wave-by-Wave Build Plan

> Detailed execution plan across all 5 waves. What gets built, when, by whom, and what the judges evaluate.

---

## Program Timeline

| Wave | Build Period | Evaluation | Grant |
|---|---|---|---|
| Wave 1 | Mar 21 – Mar 28 | Mar 28 – Mar 30 | $5,000 USDC |
| Wave 2 | Mar 30 – Apr 6 | Apr 6 – Apr 8 | $5,000 USDC |
| Wave 3 (Marathon) | Apr 8 – May 8 | May 8 – May 11 | $12,000 USDC |
| Wave 4 | May 11 – May 20 | May 20 – May 23 | $14,000 USDC |
| Wave 5 | May 23 – Jun 1 | Jun 1 – Jun 5 | $14,000 + $2,000 bonus |

---

## WAVE 1 — Foundation (Mar 21–28)
### Goal: Prove the FHE mechanism works. Ship something deployable.

This wave is about demonstrating the core FHE primitive: encrypted income accumulation and threshold verification. Everything else is secondary.

---

### What to build

#### Day 1–2: Core Contract

**`InformalProof.sol`**
- `registerWorker()` — worker onboarding
- `registerLender(address)` — lender whitelist (owner only)
- `recordIncome(InEuint64)` — encrypt and accumulate income
- `proveIncome(address, uint64)` — threshold comparison, returns `ebool`
- `linkEscrow(bytes32, address, uint64)` — link escrow to worker
- `resetMonthlyIncome()` — monthly reset with 30-day guard
- Full ACL model: `FHE.allowThis` + `FHE.allow(worker)` + `FHE.allow(lender)`

**`InformalProofGate.sol`**
- Implements `IConditionResolver` from ReinieraOS
- `isConditionMet(bytes32 escrowId)` → reads worker + threshold from InformalProof → calls `proveIncome()` → decrypts `ebool` → returns `bool`

#### Day 3: Tests

```
test/
├── InformalProof.test.ts
│   ├── registerWorker — success + duplicate rejection
│   ├── recordIncome — stores encrypted, ACL correct
│   ├── recordIncome — accumulates multiple correctly
│   ├── recordIncome — only registered worker can call
│   ├── proveIncome — true when income >= threshold
│   ├── proveIncome — false when income < threshold
│   ├── proveIncome — only lender can call
│   ├── proveIncome — ACL: lender + worker can see, public cannot
│   ├── resetMonthlyIncome — works after 30 days
│   └── resetMonthlyIncome — reverts before 30 days
├── InformalProofGate.test.ts
│   ├── isConditionMet — returns true when income qualifies
│   ├── isConditionMet — returns false when income below threshold
│   └── isConditionMet — reverts if escrow not linked
└── integration/FullFlow.test.ts
    ├── Worker registers → records 4 weeks income → lender proves → qualifies
    └── Worker registers → records insufficient income → lender proves → fails
```

All tests use `hre.cofhe.mocks` for local execution — no testnet needed.

#### Day 4: Deploy

- Deploy `InformalProof.sol` to Arbitrum Sepolia
- Deploy `InformalProofGate.sol` to Arbitrum Sepolia
- Verify both contracts on Arbiscan
- Register one test lender address
- Run manual end-to-end: `registerWorker` → `recordIncome` × 4 → `proveIncome`
- Confirm CoFHE events visible in block explorer

#### Day 5–6: Frontend (React demo dApp)

**Worker view:**
- Connect wallet (WalletConnect / injected)
- Register as worker
- Simulate income from "RappiBank" — enter amount, encrypt, submit
- View own income (decrypted locally via `decryptForView`)
- Apply for credit — calls `proveIncome` via lender test account

**Lender view:**
- Register as lender (owner-controlled in Wave 1)
- Input worker address + income threshold
- Call `proveIncome` → see `ebool` result
- See "qualifies ✅" or "does not qualify ❌"
- Never sees the income amount at any step

#### Day 7: Documentation

Three required documents:
1. `README.md` — what it is, why it exists, how to run it
2. `ARCHITECTURE.md` — contracts, FHE operations, ACL model, data flows
3. `FHE_EXPLAINER.md` — plain English: what is FHE, why it's needed here

---

### Wave 1 Deliverables

| Deliverable | Status target |
|---|---|
| `InformalProof.sol` deployed on Arbitrum Sepolia | ✅ |
| `InformalProofGate.sol` deployed on Arbitrum Sepolia | ✅ |
| Contract verified on Arbiscan | ✅ |
| All tests passing with CoFHE mocks | ✅ |
| React frontend — worker flow | ✅ |
| React frontend — lender verification flow | ✅ |
| README + ARCHITECTURE + FHE_EXPLAINER | ✅ |
| GitHub repo public with full docs | ✅ |

---

### Wave 1 Demo Script (3-minute version)

```
0:00 — The problem
"47 million informal workers in LATAM earn real income in stablecoins 
but can't prove it to a lender without exposing everything."

0:30 — Worker registers and records income
[Show: registerWorker() tx · recordIncome() × 4 txs · encrypted on-chain]

1:00 — Worker sees own income
[Show: decryptForView → income appears ONLY in the UI, not on-chain]

1:30 — Lender requests proof
[Show: proveIncome() call · ebool result · lender sees "qualifies ✅"]

2:00 — What the lender NEVER sees
[Show: block explorer · euint64 ciphertext · no amount visible]

2:30 — Why FHE and not ZK or centralized
[30 seconds on the comparison]

3:00 — Roadmap
"Wave 2: Privara integration + AI advisor + full escrow flow"
```

---

### Wave 1 Success Criteria

- [ ] Contracts deployed and verified on Arbitrum Sepolia
- [ ] Full flow works on testnet: register → record → prove
- [ ] Frontend shows worker income without leaking it to lender
- [ ] Tests cover all core functions with mocks
- [ ] README clearly explains why FHE is non-negotiable here

---

## WAVE 2 — AI + Privara + Escrow (Mar 30 – Apr 6)
### Goal: Complete the user experience. Integrate ReinieraOS. Ship the AI advisor.

---

### What to build

#### Privara / ReinieraOS Income Integration
- Replace manual income entry with `@reineira-os/sdk` listener
- `reineira.onPaymentReceived()` → automatically calls `recordIncome()`
- Add viem fallback: read USDC Transfer events for workers not on Privara
- Test both income sources end-to-end on Arbitrum Sepolia

#### ReinieraOS Escrow Integration
- Deploy `ConfidentialEscrow` (ReinieraOS) and link to `InformalProofGate`
- Full loan creation flow: lender creates escrow → links to worker → sets terms
- `isConditionMet()` called by escrow → triggers `proveIncome()` → funds release
- Silent failure pattern confirmed: failed condition → transfers 0, not reverts

#### ProtectionPool First Integration
- Co-design with ReinieraOS team
- Basic pool setup: stakers deposit → borrower loan backed by coverage
- FHE risk score calculation on encrypted income data
- Premium calculation and collection

#### AI Financial Advisor (WebLLM)
- Integrate `@mlc-ai/web-llm` with `Llama-3.2-3B-Instruct-q4f32_1-MLC`
- `decryptForView` → decrypted income → WebLLM inference → advice
- Loading UX: model download progress bar
- Suggested prompts for workers who don't know what to ask
- Zero server calls confirmed — local inference only

#### UX Improvements
- Mobile-responsive design
- Spanish and Portuguese UI (primary LATAM languages)
- Income history chart (transaction count + trend — no amounts)
- Loan status tracking for active loans
- ZeroDev integration begins (social login for new users)

---

### Wave 2 Deliverables

| Deliverable | Status target |
|---|---|
| Privara income auto-capture working | ✅ |
| USDC on-chain fallback working | ✅ |
| Full escrow loan flow on testnet | ✅ |
| ProtectionPool v1 deployed | ✅ |
| AI advisor working with local data | ✅ |
| Mobile-responsive UI | ✅ |
| Spanish/Portuguese language support | ✅ |
| ZeroDev social login (beta) | ✅ |

---

## WAVE 3 — Protocol + SDK Marathon (Apr 8 – May 8)
### Goal: Make InformalProof integrable. Publish the SDK. Open to fintechs.

This is the longest wave — 30 days — and the highest grant. Focus shifts from demo to protocol.

---

### What to build

#### @informalproof/sdk (npm package)
```typescript
// Target API for fintechs
import { InformalProofClient } from '@informalproof/sdk'

const client = new InformalProofClient({
  network: 'arbitrum',
  cofheConfig: { ... }
})

// Record income from any source
await client.recordIncome({ amount: 500_000000n }) // 500 USDC

// Prove income to a lender
const result = await client.proveIncome({
  worker: workerAddress,
  threshold: 400_000000n
})

// Create loan application
const loan = await client.applyForLoan({
  amount: 300_000000n,
  durationDays: 90,
  lenderId: '0x...'
})
```

#### Multi-Currency Support
- USDC (Arbitrum)
- USDT (Arbitrum)
- BRLA (Polygon) — Brazilian Real
- MXNe (Base) — Mexican Peso
- Unified income accumulation across currencies with FHE.add()

#### Compliance Layer
- AML threshold checks without revealing amounts
- `FHE.lte(monthlyIncome, amlLimit)` → `ebool` for regulator
- Regulators receive "within limits: yes/no" — no amounts
- KYC integration hook for fintechs that require it

#### Lender Dashboard
- Portfolio view with aggregate statistics only
- Per-loan status (active, repaid, defaulted) — no individual borrower data
- Risk exposure by pool — encrypted totals only
- Revenue tracking from premiums (stakers only)

#### ProtectionPool v2
- Open economy: other protocols can buy coverage from InformalProof pools
- Premium marketplace — best risk models attract most volume
- Token incentives for early stakers
- `judge()` integration fully tested for default resolution

#### Platform Modules (ReinieraOS)
- iOS app integration via ReineiraOS platform-modules
- Web app with full ZeroDev onboarding
- One-tap loan application for workers
- Push notifications for income recording prompts

#### Pilot Outreach
- Bitso developer program application
- Nequi partnership discussion
- Documentation for fintech integration in Spanish/English

---

### Wave 3 Deliverables

| Deliverable | Status target |
|---|---|
| `@informalproof/sdk` published on npm | ✅ |
| Multi-currency support (USDC, USDT, BRLA, MXNe) | ✅ |
| Compliance AML layer | ✅ |
| Lender dashboard | ✅ |
| ProtectionPool v2 with open economy | ✅ |
| iOS app (platform-modules) | ✅ |
| At least 1 fintech integration discussion started | ✅ |
| Full test coverage including integration tests | ✅ |

---

## WAVE 4 — Production Ready (May 11–20)
### Goal: Mainnet deployment. Security audit. First real integration.

---

### What to build

#### Mainnet Deployment
- Deploy `InformalProof.sol` to Arbitrum One (mainnet)
- Deploy `InformalProofGate.sol` to Arbitrum One
- Deploy `ConfidentialEscrow` and `ProtectionPool` to Arbitrum One
- Multi-sig ownership for all contracts
- Timelock on upgradeable components

#### Security Audit
- Audit of FHE operations and ACL model in `InformalProof.sol`
- Audit of gate logic in `InformalProofGate.sol`
- Audit of protection pool premium and payout logic
- Gas optimization review — batch FHE operations where possible
- Formal verification of ACL correctness

#### First Real Fintech Integration
- Target: one fintech partner embedding `@informalproof/sdk`
- Integration support from both InformalProof and ReinieraOS teams
- Co-marketing announcement

#### Performance Optimization
- Batch `recordIncome()` calls — multiple incomes in one transaction
- Gas benchmarking on Arbitrum mainnet (different from testnet)
- WebLLM model optimization — smaller model option for slower devices

---

### Wave 4 Deliverables

| Deliverable | Status target |
|---|---|
| Mainnet deployment on Arbitrum One | ✅ |
| Security audit completed | ✅ |
| Multi-sig + timelock on contracts | ✅ |
| First fintech integration (even in beta) | ✅ |
| Gas optimization report | ✅ |

---

## WAVE 5 — Launch (May 23 – Jun 5)
### Goal: Real users. Public launch. NY Tech Week presentation.

---

### What to build

#### Public Beta Launch
- Public launch of borrower-facing app
- First 1,000 workers onboarded via fintech partner
- Real loans: small amounts (max $100) for risk management
- Real ProtectionPool covering actual loans

#### NY Tech Week Presentation
- Live demo with real transactions on Arbitrum mainnet
- Borrower applies → income verified via FHE → loan disbursed via ReinieraOS
- Full flow visible in real time on Arbiscan
- No amounts visible on-chain — only ciphertexts

#### @informalproof/sdk v1.0
- Stable API, documented, versioned
- Integration guides in Spanish, Portuguese, English
- Developer portal with testnet sandbox
- Changelog and migration guides

#### Co-Marketing
- Joint announcement with ReinieraOS/Privara
- Fhenix ecosystem showcase
- Case study published by ReinieraOS

#### VC Introductions (via ReinieraOS partnership)
- Pitch deck finalized (reineira-atlas assisted)
- Warm introductions to aligned VCs
- LATAM-focused investor conversations

---

### Wave 5 Deliverables

| Deliverable | Status target |
|---|---|
| Public beta with real users | ✅ |
| Presentation at NY Tech Week | ✅ |
| `@informalproof/sdk` v1.0 published | ✅ |
| First 1,000 workers onboarded | ✅ |
| Co-marketing announcement published | ✅ |
| VC conversations initiated | ✅ |

---

## Risk Register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| CoFHE gas costs higher than expected | Medium | Wave 1 delay | Benchmark early, batch operations |
| WebLLM too slow on mobile devices | Medium | Wave 2 UX issue | Smaller model fallback option |
| ReinieraOS platform-modules delayed beyond Wave 2 | Low | Wave 2 UX delay | Build own minimal UI as fallback |
| Fintech integration takes longer than Wave 3 | High | Go-to-market delay | Focus on demo with simulated fintech |
| Security audit finds critical issues | Low | Wave 4 delay | Start audit early in Wave 3 |
| Arbitrum mainnet gas different from testnet | Medium | Wave 4 optimization | Test gas on Arbitrum One testnet |

---

## Team Responsibilities

| Role | Responsibility |
|---|---|
| Smart Contract Dev | `InformalProof.sol`, `InformalProofGate.sol`, tests, deployment |
| Frontend Dev | React app, CoFHE hooks, WebLLM integration |
| ReinieraOS co-build | `ConfidentialEscrow`, `ProtectionPool`, `platform-modules` |
| Product | Demo script, README, user flows, pitch |

---

## Tools and Workflow

| Tool | Purpose |
|---|---|
| Hardhat + @cofhe/hardhat-plugin | Contract development and local testing |
| Neo FHE AI Assistant | FHE contract code assistance |
| reineira-code | Solidity plugin generation for ReinieraOS |
| reineira-atlas | Strategy, compliance, investor readiness |
| Arbiscan | Contract verification and transaction monitoring |
| GitHub | Version control and public repo |

---

*InformalProof | Built on Fhenix CoFHE | Powered by Privara + ReinieraOS*
