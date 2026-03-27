# Lendi x ReinieraOS User Journeys and Data Flows

> Complete flows for every actor: what each user does, what data moves, where it goes, and what nobody ever sees.

---

## The Five Actors (Prioritized by Go-to-Market)

```
PRIMARY USERS (Waves 1-4):
BORROWER          SMALL LENDER         PROTECTION POOL     AI ADVISOR      PROTOCOL
(informal         (community credit    (stakers who        (WebLLM         (Lendi
 worker)           circle, P2P          fund coverage)      local model)    + ReinieraOS)
                   individual)

FUTURE USERS (Wave 5+, once traction is proven):
                  LARGE FINTECH
                  (Nequi, Rappi,
                   neobanks)
```

**Why this order matters:**
- Small lenders validate PMF in days, not months
- Individual users scale faster than enterprise contracts
- Large fintech deals require warm intros — easier once we have traction
- Build momentum with segments that move fast, then large players come to us

---

## PHASE 0 — First-Time Setup

### Borrower Onboarding

```
Borrower opens app for the first time
        ↓
Social login (Google / email via ZeroDev)
        ↓
ZeroDev creates smart account automatically
  No seed phrase generated
  No ETH needed for gas
  No wallet extension required
        ↓
Lendi.registerWorker()
  Emits: WorkerRegistered(workerAddress)
  Stores: registeredWorkers[worker] = true
  Gas: sponsored by protocol (ERC-2771)
        ↓
App checks income sources:

  SOURCE A — Privara / ReinieraOS
  Worker already receives payments via Privara?
  → @reineira-os/sdk listens for incoming payments automatically
  → No action required from worker

  SOURCE B — On-chain USDC
  Worker receives direct USDC transfers?
  → viem reads Transfer events on Arbitrum Sepolia
  → No API keys, no permissions needed

        ↓
Worker setup complete
Historial starts building on next payment received
```

**Data state after Phase 0:**

| Data | Location | Who can see |
|---|---|---|
| Worker address | On-chain (public) | Everyone — but it's just an address |
| `registeredWorkers[worker] = true` | On-chain (public) | Everyone |
| `monthlyIncome[worker]` | On-chain as `euint64` (ciphertext) | Nobody — uninitialized ciphertext |

---

### Lender Onboarding (Small Lenders — Self-Service)

```
Small lender (community credit circle or P2P individual) opens web app
        ↓
Social login (Google / email via ZeroDev)
        ↓
Self-service registration:
  Lender submits application (name, region, expected loan volume)
  Auto-approved for small loan limits (e.g., max $5K total exposure)
        ↓
Owner calls Lendi.registerLender(lenderAddress)
  Emits: LenderRegistered(lenderAddress)
  Stores: registeredLenders[lender] = true
        ↓
Lender deposits USDC to ProtectionPool
  Small amount OK (as low as $500 to start)
  Liquidity available for loans
  Earns premiums from covered loans
        ↓
Lender accesses loan marketplace
  Can fund individual loans or set auto-approval criteria
```

**Note:** Large fintech lenders (Wave 5+) go through manual KYC and higher deposit requirements. Small lenders get fast self-service onboarding to validate PMF quickly.

---

## PHASE 1 — Income Accumulation

### Flow: Payment Received via Privara

```
Employer / client pays worker in USDC via Privara
        ↓
ReinieraOS processes settlement on Arbitrum Sepolia
        ↓
ERC-20 Transfer event emitted on-chain:
  Transfer(from: employer, to: worker, value: 500_000000)
  (This event is PUBLIC — anyone can see it)
        ↓
@reineira-os/sdk detects the event:
  client.onPaymentReceived(workerAddress, callback)
        ↓
ENCRYPTION STEP — happens on worker's device
  cofheClient.encryptInputs([Encryptable.uint64(500_000000)])
  → returns InEuint64 (encrypted value + ZK proof of validity)
  → plaintext 500_000000 is GONE after this step
        ↓
Contract call: Lendi.recordIncome(InEuint64)
        ↓
ON-CHAIN FHE OPERATIONS:
  euint64 amount = FHE.asEuint64(encAmount)
    → TaskManager emits FHE operation request
    → FheOS executes: converts input to euint64 handle
    → Result: euint64 handle stored

  monthlyIncome[worker] = FHE.add(monthlyIncome[worker], amount)
    → TaskManager emits FHE.add request
    → FheOS executes: adds two ciphertexts homomorphically
    → Neither value is decrypted during addition
    → Result: new euint64 handle stored

  FHE.allowThis(monthlyIncome[worker])
    → Contract retains handle for future transactions
    → CRITICAL: if omitted, next tx cannot reference this handle

  FHE.allow(monthlyIncome[worker], worker)
    → Worker's address added to ACL for this handle
    → Worker can now call decryptForView on this handle

  txCount[worker] = FHE.add(txCount[worker], FHE.asEuint64(1))
  FHE.allowThis(txCount[worker])
        ↓
Emits: IncomeRecorded(workerAddress, timestamp)
  (timestamp is public — amount is NOT in this event)
```

**Data state after one income record:**

| Data | Location | Value | Who can see |
|---|---|---|---|
| Transfer event | On-chain public | `from, to, 500000000` | Everyone |
| `monthlyIncome[worker]` | On-chain `euint64` | Encrypted ciphertext | Nobody in plaintext |
| `txCount[worker]` | On-chain `euint64` | Encrypted (1) | Nobody in plaintext |
| IncomeRecorded event | On-chain public | `workerAddress, timestamp` | Everyone — no amount |

---

### Flow: Direct USDC Transfer (Fallback)

```
Client sends 300 USDC directly to worker wallet
        ↓
ERC-20 Transfer event on Arbitrum Sepolia:
  Transfer(from: client, to: worker, value: 300_000000)
        ↓
viem reads all incoming transfers for this month:
  client.getLogs({
    event: Transfer,
    args: { to: workerAddress },
    fromBlock: startOfMonth
  })
        ↓
totalIncome = sum of all transfer values
        ↓
ENCRYPTION STEP — device only
  cofheClient.encryptInputs([Encryptable.uint64(totalIncome)])
        ↓
Lendi.recordIncome(InEuint64)
  → same FHE operations as Source A
```

---

## PHASE 2 — AI Financial Advisor

### Flow: Worker Opens AI Chat

```
Worker taps "Financial Advice" in app
        ↓
DECRYPTION STEP — device RAM only
  const { decryptedValue } = await cofheClient
    .decryptForView(monthlyIncomeHandle, FheTypes.Uint64)
    .execute()

  Behind the scenes:
  1. cofheClient sends decrypt request with worker's permit
  2. Threshold Network receives request
  3. MPC: key fragments combined by multiple nodes
  4. Decrypted value returned ONLY to the requesting client
  5. decryptedValue = 1200_000000 (1200 USDC) exists in browser RAM

  → decryptedValue NEVER sent to any server
  → decryptedValue NEVER stored to disk
  → decryptedValue exists ONLY in this JavaScript variable
        ↓
WebLLM model loaded in browser (one-time ~2GB download, cached)
  → CreateMLCEngine('Llama-3.2-3B-Instruct-q4f32_1-MLC')
  → Runs on device GPU via WebGPU
  → Zero network calls during inference
        ↓
Worker asks: "Can I afford a $300 loan over 3 months?"
        ↓
Prompt constructed locally:
  "My verified monthly income is $1200 USDC.
   I want to apply for a $300 loan at 8% APR over 3 months.
   Monthly payment would be $105. Is this advisable?"
        ↓
WebLLM generates response locally (2–8 seconds)
        ↓
Response shown in chat UI
        ↓
Session ends / tab closes
  → decryptedValue garbage collected from RAM
  → No persistence, no logs, no traces
```

**Data state during AI session:**

| Data | Location | Who can see |
|---|---|---|
| `decryptedValue` (1200 USDC) | Browser RAM | Worker only — on their device |
| Prompt text | Browser RAM | Worker only |
| AI response | Browser RAM | Worker only |
| WebLLM model | Browser cache | Local only — not personalized |
| Any server logs | None | Nobody — no server called |

---

## PHASE 3 — Loan Application

### Flow: Borrower Applies

```
Worker: "Apply for $300 loan, 90 days"
        ↓
App calls LendiGate.isConditionMet(escrowId)

  Gate calls Lendi.proveIncome(worker, 400_000000)
  [threshold: $400/month minimum]

  ON-CHAIN FHE OPERATIONS:
    euint64 required = FHE.asEuint64(400_000000)
    ebool qualifies = FHE.gte(monthlyIncome[worker], required)
      → TaskManager emits FHE.gte request
      → FheOS compares TWO ciphertexts homomorphically
      → Neither income NOR threshold is decrypted during comparison
      → Result: ebool handle (encrypted true or false)

    FHE.allow(qualifies, lender)   → lender can decrypt result
    FHE.allow(qualifies, worker)   → worker can see result too

  Emits: ProofRequested(lender, worker)
  [No amounts in this event]
        ↓
Gate decrypts ebool for condition check:
  (uint256 result, bool valid) = FHE.getDecryptResultSafe(qualifies)

        ┌────────────────────────────────┐
        │  result == 1 (qualifies: true) │──→ continue to Phase 4
        │  result == 0 (qualifies: false)│──→ Phase 3B: Silent failure
        └────────────────────────────────┘
```

### Phase 3B: Silent Failure (does not qualify)

```
Gate returns false → ConfidentialEscrow.redeem() called
        ↓
FHE.select(condition=false, paidAmount, zero)
  → selects zero instead of paidAmount
  → transfers 0 USDC to worker
  → DOES NOT REVERT
  → Does not reveal WHY it failed:
    - Was income below threshold? Unknown.
    - Was pool empty? Unknown.
    - Was there a technical error? Unknown.
        ↓
Worker sees: "Not eligible at this time"
Lender sees: Nothing — the escrow simply did not release
Nobody knows the reason
```

---

### Flow: Protection Pool Premium Calculation (if qualifies)

```
ebool qualifies = true
        ↓
ProtectionPool.calculatePremium(worker, loanAmount)

  ON-CHAIN FHE OPERATIONS (ReinieraOS):
    FHE risk score computed on encrypted income history
    Premium = riskScore × loanAmount / 100
    → Result: euint64 premiumAmount (encrypted)
    → FHE.allow(premiumAmount, worker)
    → FHE.allow(premiumAmount, lender)
        ↓
Premium decrypted for display:
  Worker sees: "Coverage fee: $6"
  Lender sees: "Premium collected: $6"
        ↓
Worker confirms terms (off-chain EIP-712 signature)
  No gas required for signature
  Terms: amount=$300, duration=90days, APR=8%, premium=$6
        ↓
Pool confirms coverage active
```

---

## PHASE 4 — Loan Disbursement

### Flow: Lender Funds Escrow → Funds Released

```
Lender deposits $300 USDC to ConfidentialEscrow
        ↓
ConfidentialEscrow stores encrypted:
  Escrow {
    owner:      encrypt(workerAddress)   → eaddress
    caller:     encrypt(lenderAddress)   → eaddress
    amount:     encrypt(300_000000)      → euint64
    paidAmount: encrypt(300_000000)      → euint64
    isRedeemed: encrypt(false)           → ebool
    exists:     true                     → bool (only public field)
  }
        ↓
Escrow checks 3 gate conditions simultaneously:

  GATE 1 — LendiGate.isConditionMet(escrowId)
    → proveIncome(worker, threshold)
    → FHE.gte → ebool → true ✅

  GATE 2 — Both parties confirmed terms
    → Worker EIP-712 signature verified ✅

  GATE 3 — ProtectionPool coverage active
    → Pool has sufficient liquidity ✅

  All gates pass:
        ↓
FHE.select(allConditionsMet=true, paidAmount, zero)
  → selects paidAmount
        ↓
Worker smart account receives 294 USDC
  ($300 loan - $6 premium)
        ↓
Emits: EscrowRedeemed(escrowId)
  [No amounts in event]
        ↓
If cross-chain disbursement needed (e.g., lender on Ethereum):
  Circle CCTP V2 activated
  Operator relay: burn on source → mint on Arbitrum
  3-tier fallback ensures completion:
    0–60s:   assigned operator
    60–600s: any staked operator
    600s+:   anyone can relay
```

---

## PHASE 5 — Repayment

### Flow: Monthly Repayment

```
Month 1: Worker repays $105 USDC
        ↓
Worker sends $105 to ConfidentialEscrow on-chain
        ↓
Escrow records payment (encrypted):
  paidBack[escrow] = FHE.add(paidBack[escrow], encrypt(105_000000))
  FHE.allowThis(paidBack[escrow])
        ↓
Lender receives $105 USDC automatically
  FHE.select(paymentReceived, installmentAmount, zero)
        ↓
Lendi updates worker history:
  repaymentHistory[worker] = FHE.add(repaymentHistory[worker], encrypt(1))
  → Good repayment record (encrypted count, not amounts)
  → Future FHE risk score improves
        ↓
Emits: InstallmentPaid(escrowId, month)
  [No amount in event]
        ↓
Months 2 and 3: same flow
        ↓
Month 3 complete:
  isRedeemed → FHE.select → true
  Escrow closed
  Worker's credit history improved
```

---

## PHASE 6 — Default Resolution

### Flow: Missed Payment

```
Payment deadline passes — no payment received
        ↓
Escrow detects missed payment (on-chain timestamp check)
        ↓
ProtectionPool.activateClaim(escrowId)
        ↓
judge() called:
  FHE evaluates proof of missed payment
  → Checks encrypted payment record
  → Compares against expected installment schedule
  → Returns encrypted verdict: ebool defaultConfirmed
  → NO manual process, NO human review
        ↓
FHE.allow(verdict, pool)
FHE.allow(verdict, lender)
        ↓
Pool decrypts verdict for claim processing
        ↓
        ┌─────────────────────────────────────────┐
        │  Pool has sufficient liquidity          │
        │  FHE.select(hasFunds, remaining, avail) │
        │  → Lender receives full principal       │──→ Loan closed
        │  → Borrower marked as defaulted         │
        └─────────────────────────────────────────┘
        OR
        ┌─────────────────────────────────────────┐
        │  Pool partially exhausted               │
        │  → Lender receives available amount     │
        │  → Remainder queued for next premiums   │──→ Partial payout
        │  → Pool isolated (other pools unaffected│
        └─────────────────────────────────────────┘
        ↓
Worker's encrypted credit history updated:
  defaultCount[worker] = FHE.add(defaultCount[worker], FHE.asEuint64(1))
  → Future loans have higher encrypted risk score
  → Lender never knows exact default count — only sees higher premium
```

---

## Complete Privacy Map — What Each Actor Sees at Every Phase

```
PHASE            BORROWER                LENDER                  ON-CHAIN OBSERVER
─────────────────────────────────────────────────────────────────────────────────────
Phase 0          Own address             Own address             Both addresses (public)
Setup            Registered status       Registered status       Registered flags

Phase 1          Own income (RAM only)   Nothing                 euint64 ciphertext
Income           Own tx count (UI)       Nothing                 Timestamp of recording

Phase 2          Own income (RAM, temp)  Nothing                 Nothing
AI Advisor       AI responses (RAM)      Nothing                 Nothing

Phase 3          "Qualifies: ✅ or ❌"   "Qualifies: ✅ or ❌"   ebool ciphertext
Loan Request     Premium amount          Premium amount          escrowId created

Phase 4          294 USDC received       300 USDC deposited      Escrow exists (bool)
Disbursement     Loan terms              Loan terms              No amounts visible

Phase 5          Own payments sent       Installments received   Timestamps
Repayment        Loan balance            Loan balance            No amounts

Phase 6          "Defaulted" status      Principal recovered     Claim event (no amounts)
Default          Updated risk score      Via protection pool     No individual data
```

---

## Data Lifecycle — Where Every Sensitive Value Goes

| Value | Created | Encrypted | Stored | Revealed | Destroyed |
|---|---|---|---|---|---|
| Income amount | Worker's device (browser) | Before leaving device | `euint64` on Arbitrum | Never in plaintext | Plaintext gone after encryption |
| Monthly total | Device (RAM) | Before contract call | `euint64` on Arbitrum | Only to worker via `decryptForView` | RAM cleared after session |
| Qualification result | On-chain FHE | Already encrypted | `ebool` on-chain | Boolean to lender + worker | Never full amount |
| Loan amount | Lender's device | Before escrow creation | `euint64` in escrow | To lender + borrower | Never public |
| AI advisor data | `decryptForView` output | Decrypted in RAM | NOT stored anywhere | Worker only, RAM only | Tab close / session end |
| Risk score | ProtectionPool FHE | Always encrypted | Pool state | Never — only premium result | N/A |
| Pool balance | Pool contract | Always encrypted | `euint64` in pool | Never — prevents bank runs | N/A |

---

## Event Log — What Is Public On-Chain

All events emitted by Lendi contain NO sensitive amounts:

| Event | Public fields | Private fields |
|---|---|---|
| `WorkerRegistered(worker)` | Worker address | — |
| `LenderRegistered(lender)` | Lender address | — |
| `IncomeRecorded(worker, timestamp)` | Worker address, timestamp | Income amount (encrypted) |
| `ProofRequested(lender, worker)` | Both addresses | Neither income nor threshold |
| `EscrowLinked(escrowId, worker)` | escrowId, worker | Threshold amount |
| `MonthlyReset(worker, timestamp)` | Worker address, timestamp | Previous balance |

---

## Error Handling

| Error | Cause | Behavior | Privacy impact |
|---|---|---|---|
| `ACLNotAllowed` | Missing `FHE.allowThis()` after operation | Transaction reverts | None — no data leaked |
| Gate returns false | Worker does not qualify | `FHE.select → 0` transfer | None — reason hidden |
| Pool exhausted | Insufficient coverage liquidity | Partial payout, queue remainder | None — balances encrypted |
| CCTP attestation delay | Circle service unavailable | Settlement paused, auto-resumes | None — funds not at risk |
| Operator offline | Relay not submitted | 3-tier fallback activates | None — eventually resolved |
| Decrypt result not ready | CoFHE processing delay | `require(valid)` reverts | None — retry on next block |

---

*Lendi | Built on Fhenix CoFHE | Powered by Privara + ReinieraOS*
