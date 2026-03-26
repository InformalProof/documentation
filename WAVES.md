# WAVES.md
## InformalProof — Wave-by-Wave Build Plan

> Realistic deliverables per wave. No padding. If it's not built, it's not listed.

---

## Deployment Strategy

**Fhenix mainnet is not available until autumn 2026.**

Our phased approach:
1. **Waves 1–2:** Arbitrum Sepolia (testnet) — build and validate the FHE mechanism
2. **Waves 3–4:** Public mainnet (Arbitrum One) — validate product-market fit with real users, no FHE in production yet
3. **Wave 5 + beyond:** Fhenix mainnet when available — plug in privacy as a protocol-level feature with Fhenix support

This means our go-to-market is not blocked by Fhenix mainnet timing. We validate the product first. Privacy becomes the upgrade.

---

## WAVE 1 — Foundation
### Mar 21–28 | Evaluation Mar 28–30 | $5,000 USDC

**Goal:** Deploy working FHE contracts. Demo the core mechanic. Ship clean docs.

### What gets built

**Contracts (Arbitrum Sepolia)**
- `InformalProof.sol`
  - `registerWorker()` + `registerLender()`
  - `recordIncome(InEuint64)` — encrypt + accumulate
  - `proveIncome(address, uint64)` → `ebool`
  - `linkEscrow(bytes32, address, uint64)` — gate setup
  - `resetMonthlyIncome()` — 30-day guard
  - Full ACL: `allowThis` + `allow(worker)` + `allow(lender)`
- `InformalProofGate.sol`
  - Implements `IConditionResolver`
  - `isConditionMet(escrowId)` → calls `proveIncome` → returns `bool`

**Tests (Hardhat + CoFHE mocks)**
- `registerWorker` — success + duplicate rejection
- `recordIncome` — encrypts correctly, ACL set right
- `recordIncome` — accumulates across multiple calls
- `recordIncome` — only registered worker can call
- `proveIncome` — true when income >= threshold
- `proveIncome` — false when income < threshold
- `proveIncome` — only lender can call
- `proveIncome` — ACL: lender + worker can decrypt, public cannot
- `resetMonthlyIncome` — works after 30 days, reverts before
- Gate: `isConditionMet` — true / false / reverts if not linked
- Integration: full worker → income × 4 → prove → result

**Minimum: 12 passing tests.**

**Frontend (React — deployed on Vercel)**
- Worker view: register, simulate income entry, view own balance (decrypted locally)
- Lender view: input worker address + threshold → see ebool result
- Demo makes clear: lender sees only ✅ or ❌, never the number

**Documentation**
- `README.md` — pitch deck format (this file)
- `PRODUCT.md` — personas, flows, differentiators
- `ARCHITECTURE.md` — Mermaid diagrams + contracts
- `WAVES.md` — this file

### What does NOT get built in Wave 1
- ❌ Privara integration (manual income entry only)
- ❌ ReinieraOS escrow integration
- ❌ AI advisor
- ❌ ZeroDev accounts
- ❌ ProtectionPool
- ❌ Mobile UI

### Demo script (3 minutes)

```
0:00 — The problem
"Sebastián is a Rappi courier in Medellín.
He earns $350/month in USDC. Addi rejected him — no payslip.
He has income. The system can't see it without violating his privacy."

0:40 — Worker records income
[Show: recordIncome() × 4 txs on Arbitrum Sepolia]
[Show: Arbiscan — euint64 ciphertext, no amount visible]

1:10 — Worker sees own balance
[Show: decryptForView → income appears in UI only]
[Explain: this runs in browser RAM, never sent anywhere]

1:40 — Lender requests proof
[Show: proveIncome() → ebool → "qualifies ✅"]
[Show: lender UI has NO income number anywhere]

2:20 — The key point
"The lender answered their question without seeing the data.
This cannot be built with ZK or any centralized system.
Only FHE allows computation on encrypted state that changes over time."

2:50 — What's next
"Wave 2: Privara income capture + ReinieraOS escrow + AI advisor.
Colombia first. Nequi integration target by Q3."
```

---

## WAVE 2 — Integration
### Mar 30 – Apr 6 | Evaluation Apr 6–8 | $5,000 USDC

**Goal:** Automate income capture. Add escrow flow. Ship AI advisor.

### What gets built

**Privara income integration**
- `@reineira-os/sdk` listener — auto-captures incoming payments
- viem fallback — reads USDC Transfer events directly
- Both sources tested end-to-end on Arbitrum Sepolia

**ReinieraOS escrow integration (co-build with Reineira team)**
- `ConfidentialEscrow` linked to `InformalProofGate`
- Full loan creation flow: lender creates escrow → links worker → sets threshold
- Silent failure confirmed: failed condition → transfers 0, does not revert
- Basic ProtectionPool v1: stakers deposit → loan backed

**AI financial advisor**
- WebLLM loaded in browser (`Llama-3.2-3B-Instruct-q4f32_1-MLC`)
- `decryptForView` → income in RAM → WebLLM inference
- Loading progress bar (first load ~2GB, then cached)
- 3 suggested prompts for workers who don't know what to ask
- Confirmed: zero server calls during inference

**UX**
- Mobile-responsive layout
- Spanish UI (primary market: Colombia)
- ZeroDev social login beta (email / Google → smart account)

### What does NOT get built in Wave 2
- ❌ Multi-currency (USDC only)
- ❌ Full ProtectionPool (v1 only — basic coverage)
- ❌ npm SDK for fintechs
- ❌ iOS app

---

## WAVE 3 (Marathon) — Protocol
### Apr 8 – May 8 | Evaluation May 8–11 | $12,000 USDC

**Goal:** Make it integrable. First real fintech contact.

### What gets built

**`@informalproof/sdk` — npm package**
```typescript
import { InformalProofClient } from '@informalproof/sdk'

const client = new InformalProofClient({ network: 'arbitrum' })
await client.recordIncome({ amount: 500_000000n })
const result = await client.proveIncome({ worker, threshold: 400_000000n })
```

**Compliance layer**
- AML threshold check: `FHE.lte(monthlyIncome, amlLimit)` → `ebool` for regulator
- Regulator receives "within limits: yes/no" — no amounts

**ProtectionPool v2**
- `judge()` integration — default resolution without manual process
- Open economy: other protocols can buy coverage from our pools

**Colombia pilot outreach**
- Nequi developer program application
- Rappi Colombia developer contact
- Clear integration guide in Spanish

**Public mainnet preparation**
- Deploy to Arbitrum One (mainnet) — without FHE in production
- Validate loan flow with real USDC
- Collect real user feedback before Fhenix mainnet

---

## WAVE 4 — Validation
### May 11–20 | Evaluation May 20–23 | $14,000 USDC

**Goal:** Real users. Real loans. Colombia only.

### What gets built

**First real loans on public mainnet**
- Small loans only: max $100 per borrower in pilot
- Target: 100 loans via one fintech partner (Nequi or Rappi Colombia)
- Real repayment tracking
- Real ProtectionPool covering actual defaults

**Security**
- Audit of `InformalProof.sol` and `InformalProofGate.sol`
- Multi-sig ownership on all contracts
- Gas optimization report — Arbitrum mainnet differs from testnet

**Fhenix mainnet preparation**
- Monitor Fhenix mainnet timeline (scheduled autumn 2026)
- Prepare migration path: public mainnet → Fhenix mainnet
- Co-design privacy plug-in with Fhenix team

---

## WAVE 5 — Privacy Layer
### May 23 – Jun 5 | Evaluation Jun 1–5 | $14,000 + $2,000 bonus

**Goal:** Present at NY Tech Week. Prepare for Fhenix mainnet migration.

### What gets built

**NY Tech Week presentation**
- Live demo on public mainnet with real users
- Full flow: income → proof → loan disbursed
- Show Arbiscan: what the lender sees vs. what's actually on-chain

**`@informalproof/sdk` v1.0**
- Stable API, versioned
- Integration guide in Spanish and English
- Developer sandbox on testnet

**Fhenix mainnet readiness**
- Architecture ready for privacy plug-in when mainnet ships (autumn 2026)
- Co-marketing plan with Fhenix team
- VC introductions via ReinieraOS partnership (already committed)

### What does NOT ship in Wave 5
- ❌ Fhenix mainnet — not available until autumn 2026
- ❌ Mexico or Brazil expansion — Colombia pilot must work first
- ❌ Full open economy for protection pools — needs more validation

---

## Honest Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| CoFHE gas costs too high for small loans | Medium | Batch FHE ops; test gas early Wave 1 |
| WebLLM too slow on low-end Android devices | High | Smaller model option; server fallback for Wave 2 |
| Fintech integration takes longer than Wave 3 | High | Focus on self-serve demo, not B2B in Wave 1–2 |
| Fhenix mainnet delayed past autumn 2026 | Low | Public mainnet validates product anyway |
| ProtectionPool insufficient liquidity for pilot | Medium | Start with small loan amounts only ($100 max) |

---

## Geographic Focus

**Wave 1–2:** Testing only, no geography
**Wave 3–4:** Colombia exclusively
- Medellín + Bogotá — highest gig worker density
- Nequi: 19M users, 60%+ informal workers
- Rappi Colombia: 300K+ active couriers
- LGPD-equivalent law (Habeas Data) creates compliance demand

**Wave 5+:** Mexico if Colombia pilot shows traction
- Mexico: $67B remittance corridor
- Bitso: established stablecoin infrastructure

**Not in 2026:** Brazil — market is large but requires Portuguese UI, different regulatory framework, and separate fintech relationships.

---

*InformalProof | Built on Fhenix CoFHE | Powered by Privara + ReinieraOS*
