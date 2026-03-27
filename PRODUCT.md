# Lendi Product Overview

---

## User Personas

### Persona 1 - The Worker (Borrower)

**Name:** Sebastián, 28, Medellín
**Job:** Rappi courier, 4 days/week
**Monthly income:** ~$350 USDC (received via stablecoin)
**Problem:** Needs $200 to repair his motorcycle. Addi and Juancho rejected him — no payslip. He has income but no proof the system accepts.
**What he wants:** A small loan, fast, without sending his bank statements to a stranger.
**What he doesn't want:** To share his client list, income sources, or transaction history with anyone.
**Tech level:** Uses WhatsApp and Rappi. Has a Nequi account. Has never heard of FHE and shouldn't need to.

---

### Persona 2 - The Small Lender (Primary Go-to-Market)

**Name:** María, runs a community credit circle in Medellín
**Problem:** Lends to 20-30 informal workers in her neighborhood. Knows them personally but has no way to verify their income. When someone defaults, she has no recourse.
**What she wants:** A simple way to verify borrowers can repay without needing their bank statements. Protection when loans go bad.
**What she doesn't want:** Complex fintech infrastructure. Legal liability for holding financial data.
**Why she's our starting point:** Fast to onboard, validates PMF in days not months, builds social proof before approaching big tech.

---

### Persona 3 - The Individual P2P Lender

**Name:** Independent lender providing microloans via crypto
**Problem:** Wants to lend to informal workers but has no credit scoring system. High default risk.
**What they want:** Income verification without KYC overhead. Automated default protection.
**What they don't want:** To manually chase down borrowers when payments are missed.
**Why they matter:** Volume can scale quickly if UX is good. These users become our advocates when we approach larger partners later.

---

### Persona 4 - The Large Fintech Integration Partner (Future, Not Wave 1-3)

**Name:** Product lead at Nequi Colombia
**Problem:** Has 19M users, most informal. Wants to offer credit without building FHE infrastructure.
**What they want:** Proven traction with real users before integration. An SDK that works.
**What they don't want:** To be a guinea pig for an unvalidated product.
**Why we wait:** Large tech sales take months without warm intros. We build momentum with smaller segments first, then they come to us.

---

## Core User Flows

### Flow 1 — Worker Registers and Builds Income History

```
Worker opens app (or fintech app with Lendi)
→ Social login via ZeroDev (no seed phrase, no gas)
→ registerWorker() on Lendi.sol
→ Privara SDK listens for incoming stablecoin payments
→ Each payment: encrypted client-side → recordIncome(InEuint64)
→ euint64 accumulates on Arbitrum — nobody reads it
→ Worker builds encrypted income history silently
```

### Flow 2 — Worker Checks Their Own Finances

```
Worker opens AI advisor tab
→ decryptForView() — income decrypted in browser RAM only
→ WebLLM loads locally (no server call)
→ Worker asks: "Can I afford a $200 loan?"
→ AI responds with personalized advice
→ Session closes — data cleared from RAM
→ Zero data ever left the device
```

### Flow 3 — Worker Applies for Credit

```
Worker taps "Apply for loan — $200"
→ LendiGate.isConditionMet(escrowId)
→ proveIncome(worker, threshold=200_000000)
→ FHE.gte(monthlyIncome, required) on ciphertexts
→ ebool result: qualifies ✅ or ❌
→ If qualifies: ReinieraOS escrow releases funds
→ Worker receives USDC in smart account
→ Lender never saw the income amount
```

### Flow 4 — Lender Gets Protected Against Default

```
Lender deposits liquidity to fund loans
→ ProtectionPool.stake(amount) — lender is now insured
→ Loan is disbursed to qualified worker
→ If repaid: lender receives principal + interest
→ If default: ProtectionPool.activateClaim(escrowId)
   → FHE judge() runs encrypted default logic
   → Lender receives automated payout from pool
   → No manual review, no exposure of worker data
→ Lender dashboard shows: active loans, repayment rate, pool coverage %
→ Lender NEVER sees: worker income, identity, risk score
```

**This is the feature no competitor has.** Kueski, Addi, and Juancho expose all borrower data to calculate risk. ConfidentialCredit has privacy but no lender protection. Lendi is the only platform that gives lenders coverage while keeping borrower data encrypted end-to-end.

---

## Key Features

### 1. Encrypted Income Accumulation
- Income from Privara payments or direct USDC transfers
- Encrypted client-side before touching the contract
- Accumulated via `FHE.add()` — no decryption during addition
- Monthly reset with 30-day guard

### 2. Credit Verification Without Exposure
- Lender sets threshold (e.g., $300/month minimum)
- `FHE.gte()` compares two ciphertexts — neither revealed
- Returns `ebool` only: qualifies or not
- Lender provably cannot access the income amount

### 3. Local AI Financial Advisor
- `decryptForView` decrypts income in browser RAM
- WebLLM runs inference locally — no network call
- Advice based on real income data
- Completely private: device-only computation

### 4. Encrypted Lender Protection (ProtectionPool)
- **Our only feature that no competitor has**
- Lenders deposit liquidity and are automatically insured against borrower default
- Premium calculation uses `FHE.mul()` and `FHE.div()` on encrypted risk scores — borrower data never exposed
- Default claims are processed via encrypted `judge()` logic — automated payout with no manual review
- Backed by staker deposits in a shared pool — distributed risk across all loans
- This is what makes Lendi defensible: privacy + lender coverage in one system

### 5. ReinieraOS Loan Lifecycle
- Escrow holds funds until all conditions pass
- Automatic repayment tracking
- Integration with ProtectionPool for default resolution

### 6. ZeroDev Account Abstraction
- Workers onboard with social login
- No wallet, no seed phrase, no gas management
- Smart account created automatically

---

## What Makes It Different

### The one thing no competitor does: Lender protection without exposing borrower data

According to our competitor analysis, **lender protection is our only unique feature**. Every other platform — Kueski, Addi, Juancho te Presta, even ConfidentialCredit — either has no lender protection at all, or requires full visibility into borrower finances to calculate risk.

Lendi is the only system where:
- **Lenders are insured against default via an encrypted ProtectionPool**
- **Risk premiums are calculated using FHE** — no individual borrower data is ever decrypted
- **Claims are paid automatically** when defaults occur — using encrypted `judge()` logic
- **The borrower's income and risk score remain encrypted end-to-end**

This is our moat. Privacy alone is not enough — Bloom and ConfidentialCredit have privacy. Serving informal workers is not enough — Addi and Kueski already do that. But nobody else gives lenders coverage without requiring full financial transparency from the borrower.

### Why this matters for go-to-market

**Starting with small lenders and individual users (Waves 1-4):**
- **Faster validation:** Days to first loan, not months to first enterprise contract
- **Lower friction:** Self-service onboarding, no warm intros needed
- **Real traction data:** When we approach Nequi or Rappi later, we show them "5,000 loans in 90 days" not a pitch deck
- **Community-driven growth:** Small lenders tell other small lenders. Virality beats cold outreach.

**Moving to large fintech later (Wave 5+):**
Once we have proven user demand, large fintechs care about two things: risk and compliance. Lendi reduces both:
- **Risk:** ProtectionPool coverage means the fintech is insured even if the borrower defaults
- **Compliance:** Because we never hold plaintext income data, the fintech has no data liability under LGPD or Habeas Data laws

This is not a UX advantage. It is a structural privacy and risk guarantee enforced by cryptography. As regulations tighten, this becomes a hard-to-replicate integration advantage. But we prove it with small users first.

### Why FHE specifically

ZK proofs can verify a single statement ("I earned more than $X"). They cannot maintain a mutable encrypted history that updates with each new transaction. That is exactly what income verification requires — a running encrypted balance that grows week by week.

Only FHE allows computation on encrypted state that changes over time. That is why Lendi cannot be built with ZK, TEEs, or transparent blockchains.

---

## What We Are Not Building (Scope Limits)

To be explicit about what Wave 1 does NOT include:

- ❌ Mobile app — demo dApp only in Wave 1
- ❌ Real fintech integration — simulated in Wave 1
- ❌ Mainnet deployment — Fhenix mainnet scheduled autumn 2026
- ❌ Protection pool — architecture designed, build starts Wave 2
- ❌ Multi-currency support — USDC only in Wave 1
- ❌ ZeroDev integration — Wave 2

Wave 1 is: contracts deployed, tests passing, demo working, docs complete.

---

*Lendi | Built on Fhenix CoFHE | Powered by Privara + ReinieraOS*
