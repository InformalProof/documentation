# PRODUCT.md
## InformalProof — Product Overview

---

## User Personas

### Persona 1 — The Worker (Borrower)

**Name:** Sebastián, 28, Medellín
**Job:** Rappi courier, 4 days/week
**Monthly income:** ~$350 USDC (received via stablecoin)
**Problem:** Needs $200 to repair his motorcycle. Addi and Juancho rejected him — no payslip. He has income but no proof the system accepts.
**What he wants:** A small loan, fast, without sending his bank statements to a stranger.
**What he doesn't want:** To share his client list, income sources, or transaction history with anyone.
**Tech level:** Uses WhatsApp and Rappi. Has a Nequi account. Has never heard of FHE and shouldn't need to.

---

### Persona 2 — The Lender

**Name:** Fintech credit team at a neobank
**Problem:** They want to expand credit to informal workers but have no reliable income signal. They also have data liability concerns — storing worker financial data creates LGPD / Habeas Data compliance risk.
**What they want:** A verified income signal that tells them "qualifies" or "doesn't qualify" — without holding the underlying data.
**What they don't want:** To be responsible for a database of worker income that can be hacked or subpoenaed.

---

### Persona 3 — The Integration Partner (Fintech)

**Name:** Product lead at Nequi Colombia
**Problem:** Has 19M users, most informal. Wants to offer credit without building FHE infrastructure.
**What they want:** An SDK that integrates in days, not months. A widget that works inside their app.

---

## Core User Flows

### Flow 1 — Worker Registers and Builds Income History

```
Worker opens app (or fintech app with InformalProof)
→ Social login via ZeroDev (no seed phrase, no gas)
→ registerWorker() on InformalProof.sol
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
→ InformalProofGate.isConditionMet(escrowId)
→ proveIncome(worker, threshold=200_000000)
→ FHE.gte(monthlyIncome, required) on ciphertexts
→ ebool result: qualifies ✅ or ❌
→ If qualifies: ReinieraOS escrow releases funds
→ Worker receives USDC in smart account
→ Lender never saw the income amount
```

### Flow 4 — Lender Views Portfolio

```
Lender logs into dashboard
→ Sees: active loans, repayment status, pool coverage
→ Does NOT see: any worker's income, identity, or history
→ Receives automated claim payout if default occurs
→ ProtectionPool handles resolution via encrypted judge()
```

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

### 4. ReinieraOS Loan Lifecycle
- Escrow holds funds until all conditions pass
- Automatic repayment tracking
- Default resolution via encrypted `judge()` — no manual process
- ProtectionPool covers lenders — backed by staker deposits

### 5. ZeroDev Account Abstraction
- Workers onboard with social login
- No wallet, no seed phrase, no gas management
- Smart account created automatically

---

## What Makes It Different

### The one thing no competitor does

Every micro-lending company in LATAM — Kueski, Addi, Juancho te Presta — requires the borrower to share their financial data. The platform stores it, processes it, and is liable for it.

InformalProof is the only system where:
- The lender never touches the borrower's income data
- The platform has no server storing sensitive information
- The borrower's data is mathematically protected — not just policy-protected

This is not a UX advantage. It is a structural privacy guarantee enforced by cryptography. As LGPD in Brazil and Habeas Data laws in Colombia become stricter, this becomes a compliance advantage for the fintechs that integrate us.

### Why FHE specifically

ZK proofs can verify a single statement ("I earned more than $X"). They cannot maintain a mutable encrypted history that updates with each new transaction. That is exactly what income verification requires — a running encrypted balance that grows week by week.

Only FHE allows computation on encrypted state that changes over time. That is why InformalProof cannot be built with ZK, TEEs, or transparent blockchains.

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

*InformalProof | Built on Fhenix CoFHE | Powered by Privara + ReinieraOS*
